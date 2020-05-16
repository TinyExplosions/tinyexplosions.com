---
title: It's Always DNS
date: '2020-05-16'
tags:
  - homelab
  - tech
  - blog
---
OpenShift just won’t install. I’m using IPI, which should be the foolproof method, I’m following the instructions -both in a cursory way, and also in a very detailed, read every line kind of way. I’ve kicked off the installer countless times. I’ve stared at `INFO Waiting up to 30m0s for the cluster to initialize...` for a *lot* more than 30 mins. I’ve left it overnight to see if it sorts itself. I’ve tried to debug. I’ve used up close to all the goodwill of some of my workmates (who are total champions btw).  Still though, I don’t have an installed cluster. 

During one debug session, I was sshing into a node, and tried (and failed) to do it directly via hostname. This led to some interesting experiments

```bash
$ ping ocp-nln68-master-1
PING ocp-nln68-master-1.bugcity.tech.bugcity.tech (178.0.0.11): 56 data bytes
```

What was interesting was the `.bugcity.tech.bugcity.tech`, and the fact that it was resolving to the server `178.0.0.11`, rather than the expected `178.0.0.59` This disappeared after a while, but got me digging deeper. Then something struck me. I run a Raspberry Pi with Pi-Hope on it as my network’s DNS (it calls out to cloudflare dns over https), and I use it’s built in fork of dnsmasq to do some local serving of traffic (again, I barely know what any of it does, but it seems to work). Here’s a snippet from `/etc/dnsmasq.d/01-pihole.conf`

```bash
local-ttl=2
local=/etc/hosts/
log-async
address=/.bugcity.tech/178.0.0.11
address=/.ocp.bugcity.tech/178.0.0.200
address=/.apps.ocp.bugcity.tech/178.0.0.210
server=127.0.0.1#5053
server=::1#5053
domain-needed
bogus-priv
except-interface=nonexisting
```

*short aside, once I decided to buy a lab, like most good techies, I went out and purchased a domain, `bugcity.tech` -I would use this a lot...*

I also created a nice separate network for lab based play, you know, following best practise and separating traffic and all that good stuff (again, I feel I must stress that I don't really understand *any* of this stuff deeply, so if you're an expert, feel free to roll your eyes). It was configured as below

![Screenshot of Unifi Admin console showing the networking setup](/images/network-setup.png)

Note the domain name I gave the network - `bugcity.tech`. Then, once I got my hot little fingers on the server, I fired it up (eventually) and stuck, you guessed it! `bugcity.tech` as it’s hostname, and therefore the host for RHEV. That’s a lot of work for a single name, and while I don’t have a specific evidence that this is too blame, I am highly suspicious, so things need to change. 

Another thing I investigated while all this was going on was disk speed. [According to this article](https://www.ibm.com/cloud/blog/using-fio-to-tell-whether-your-storage-is-fast-enough-for-etcd), I needed check if the 99th percentile of fdatasync durations is less than 10ms. I dutifully ran the wee tool and got, well less than that.

```bash
sync (msec): min=3, max=478, avg=20.59, stdev=36.51
    sync percentiles (msec):
     |  1.00th=[    4],  5.00th=[    6], 10.00th=[    6], 20.00th=[    9],
     | 30.00th=[   10], 40.00th=[   12], 50.00th=[   14], 60.00th=[   16],
     | 70.00th=[   17], 80.00th=[   18], 90.00th=[   20], 95.00th=[   84],
     | 99.00th=[  205], 99.50th=[  268], 99.90th=[  351], 99.95th=[  384],
     | 99.99th=[  443]
```

Things looked better on the SSD

```bash
fsync/fdatasync/sync_file_range:
    sync (usec): min=305, max=19953, avg=1602.23, stdev=720.84
    sync percentiles (usec):
     |  1.00th=[  326],  5.00th=[  347], 10.00th=[  388], 20.00th=[  586],
     | 30.00th=[ 1434], 40.00th=[ 1614], 50.00th=[ 1860], 60.00th=[ 1876],
     | 70.00th=[ 1975], 80.00th=[ 2147], 90.00th=[ 2278], 95.00th=[ 2376],
     | 99.00th=[ 2868], 99.50th=[ 2933], 99.90th=[ 3032], 99.95th=[ 4080],
     | 99.99th=[13829]
```

And looked a little better running on the final drive (the first result is 2 drives formatted as one large disk)

```bash
fsync/fdatasync/sync_file_range:
    sync (usec): min=8232, max=97259, avg=20732.36, stdev=6345.02
    sync percentiles (usec):
     |  1.00th=[11207],  5.00th=[11994], 10.00th=[12911], 20.00th=[14746],
     | 30.00th=[16581], 40.00th=[18482], 50.00th=[20317], 60.00th=[22414],
     | 70.00th=[24249], 80.00th=[25297], 90.00th=[27132], 95.00th=[33162],
     | 99.00th=[33424], 99.50th=[33424], 99.90th=[73925], 99.95th=[74974],
     | 99.99th=[91751]
```

Better, but the SSD was the only thing that beat the required 10ms. I had some interesting discussions about this on gChat, and while parts went over my head, the consensus seemed to be that this was a pretty stringent requirement, and that things should work even on the slowest partition, but to be sure I made certain the OpenShift VMs were going on the last drive, but still, no install.

This is a rambling way to say I’m starting again. Clean boot on the server -it’s a lab, they’re designed to be rebuilt on occasion, and reconfigure everything - but I’ll make changes this time.

First off, I’ll change the IP range and domain name on the network, let’s go '172' and ‘bugcity.local’.

Second, I’ll choose a better hostname for the server (maybe keep it simple and ‘server.bugcity.tech’, or just ‘server’).

Third, I’ll modify how I’m using disks. I currently have a 240GB SSD, and 3 500gb spinning disks, with the SSD set to be the boot volume. I will change this, and set one of the 500gb drives as the boot. It may mean that startup time is affected a bit, but I don’t exactly plan to reboot a lot, so I’ll deal. 

Once RHEV is back up, I’ll create one volume for VMs that is the SSD, and I’ll stripe the remaining two spinning disks as RAID-0 -and re-run `fio` to check results, but I *think* this should help eliminate disks from the equation, and *possibly* DNS.

I’ll only know for certain once I have finished. But I bet it’s DNS

It’s *always* DNS. 

