---
title: It Wan't DNS?
date: '2020-05-20'
tags:
  - homelab
  - tech
  - blog
  - DNS
  - RHEV
  - OpenShift
---
It's been a few days now since I took the steps mentioned in [my last post](/posts/its-always-dns), and I cleared everything out and started again. It took a couple of hours, but thanks to it not being the first time, and the notes I'd made on the way, I had the server (now christened `bigboy.bugcity.tech`) up, with RHEV installed, and my network and disk setup as described in the post.

Then it was time to re-run `fio`, just to see what difference had been made. First on `slow-disks` - a RAID-0 pair of 500GB spinners

```bash
fsync/fdatasync/sync_file_range:
    sync (msec): min=3, max=359, avg=27.65, stdev=15.33
    sync percentiles (msec):
     |  1.00th=[    5],  5.00th=[   12], 10.00th=[   18], 20.00th=[   20],
     | 30.00th=[   22], 40.00th=[   24], 50.00th=[   26], 60.00th=[   28],
     | 70.00th=[   30], 80.00th=[   32], 90.00th=[   42], 95.00th=[   42],
     | 99.00th=[  112], 99.50th=[  128], 99.90th=[  157], 99.95th=[  163],
     | 99.99th=[  205]
```

99th percentile was faster, at 112ms, but still way above the 10ms recommended. Lets look at `fast-disk`, the SSD

```bash
 fsync/fdatasync/sync_file_range:
    sync (usec): min=293, max=31180, avg=1507.45, stdev=722.84
    sync percentiles (usec):
     |  1.00th=[  318],  5.00th=[  334], 10.00th=[  343], 20.00th=[  379],
     | 30.00th=[ 1401], 40.00th=[ 1418], 50.00th=[ 1860], 60.00th=[ 1876],
     | 70.00th=[ 1909], 80.00th=[ 1942], 90.00th=[ 1991], 95.00th=[ 2212],
     | 99.00th=[ 2606], 99.50th=[ 2671], 99.90th=[ 2868], 99.95th=[ 2900],
     | 99.99th=[ 7439]
```

That puts the 99th percentile at 2.6ms, well within what we need. With that, I decided to run the install on the fast drive to see what happened, so did a little bastion dance, some configuration and ran `create cluster`. 40 minutes or so later....

![Dashboard of a freshly installed OpenShift Instance](/images/openshift-dashboard.png "Holy good gravy, there's an OpenShift dashboard!")

So, that told me a lot - or at least told me that it was either the networking, or the disks that were to blame. While it would be great to just move on and call it done, I can't just do that - I have to know *for sure*, so it was out with `cluster-install destroy cluster` and a reconfigure to use the slower disks.

That my friends, led to errors. Lots of timeout-y errors, the sort I have seen a lot in the last few days. This wasa the beginning of a saga that would consume most of the weekend, and lead to some interesting conclusions.

* I reinstalled everything from scratch, and attempted an install, and it worked.
* I deleted, and tried another install on a fast disk, it worked (though didn't finish cleanly, I had to reboot a master).
* I deleted, and tried on `slow-disks` and it failed.

So, that was getting somewhere - maybe it *is* disk speed that is the problem. To dig deeper, I then

* Deleted, Tried Install again on `slow-disks` - failure.
* Deleted, Tried Install on `fast-disk` - failure.
* Deleted, Tried Install on `fast-disk` - failure.
* Deleted, Tried Install on `fast-disk` - failure.

Damnit, that means I can't rule out disks for sure. It was back to a fresh install of everything, RHEL 7 on the Server, RHEV, set up a clean Bastion VM, and install on `fast-disk`. Success! Then the thought hit me, in the previous tests, after the sucessful install of OCP, I had spun up an IdM server - could that have been the problem? Only one way to find out...

* Deleted, Tried Install on `fast-disk` - failure.
* Deleted, Tried Install on `fast-disk` - failure.
* Deleted, Tried Install on `fast-disk` - failure.
* Deleted, Tried Install on `fast-disk` - failure.

Head, meet wall. This made no sense to me. I was spinning up a new VM, installing RHEL on it, and using it as a bastion each time. Deleting VMs on RHV should clear everything out, but I just wasn't seeing that happen. In a bit of desparation, I had one last idea. What if it was Unifi being an asshole with DHCP. I knew things were getting IPs ok, but maybe, just maybe.

Only thing to do was to turn off the server, and change subnet on the network. `178.0.0.x` became `170.0.0.x`. Booted up, created a bastion, ran the installer, and.... failure! THat was when I realised I'd changed *two* things instead of one. I'd also specified a new name for the cluster `lab.bugcity.tech`, not `ocp.bugcity.tech` and guess who hadn't updated that in dnsmasq of PiHole ;(

Restarted server, changed subnet, made *correct* changes in PiHole, and started the install dance again. This time - success!

So there you have it - exactly what the cause is, I don't know, but changing network settings seemed to fix things.

It looks like it was DNS.
