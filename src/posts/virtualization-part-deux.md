---
title: Virtualization - part deux
date: '2020-05-11'
tags:
  - homelab
  - tech
  - blog
---
So here we are again, back to Virtualization. As [outlined in my previous post](/posts/back-tracking), I'm ditching libvirt, and embracing the whole RHEV lifestyle (RHEV is easier to pronounce than RHV, so forgive me if I use it here and there). My starting point, as ever, was the [official dcumentation](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.3/html/installing_red_hat_virtualization_as_a_standalone_manager_with_local_databases/installing_the_red_hat_virtualization_manager_sm_localdb_deploy) - double checking to make sure they were the right ones of course :)

It seemed quite straightforward, I wanted to install the Manager and Databases all on one machine (the actual server), and so that simply required adding a couple of repos, installing `rhvm` and running `engine-setup`

```bash
subscription-manager repos --enable=rhel-7-server-rhv-4.3-manager-rpms
subscription-manager repos --enable=rhel-7-server-rhv-4-manager-tools-rpms
subscription-manager repos --enable=rhel-7-server-ansible-2.9-rpms
subscription-manager repos --enable=jb-eap-7.2-for-rhel-7-server-rpms
yum install rhvm
```

Ah, if only it was that easy. The install of `rhvm` failed, complaining that `python2-jmespath` couldn't be found, and I spent a good hour or so trying to get to the bottom of it -not helped by my lack of knowledge of all this stuff. Eventually, I took the nuclear option, modified `/etc/yum/pluginconf.d/search-disabled-repos.conf` to set `notify_only=0` and re-ran the setup. After a **long** time churning through all the repos known to man, and eventually installed correctly. Turns out our docs aren't *entirely* correct, and the `rhel-7-server-extras-rpms` repo is also needed.

Once we were installed, the setup went ok - ran `engine-setup`, to configure it on the current host, didn't install the overlay network, installed the web proxy, and got the installer to configure all the databases. It all went smoothly, and a few minutes later, I was greeted with a dashboard.

![Red Hat Virtualization Manager Dashboard](/images/rhev.png "RHEV Manager Dashboard - lots to configure.")

From there, it was time to add the first host to the system. Because I'm rolling everything into the one system, my single host is also the base install of RHEL. This time, the documents were spot on, and I soon had a host added.

Next to configure was a storage domain, for that I added a local mount made up from a couple of the drives, and ensured the default ovirtmgmt network would act as a bridge. Once complete, it was time to spin up my first VM.

### Creating VM's in RHV

There were a couple of things I found when trying to get VM's running, that might be useful for others. THe first was in getting an iso into the system to add to my machines. This is done by going to Storage -> Domains -> Domain, then selecting the 'Disks' option at the top. Then you click 'Upload -> Start' and add your iso:

![Adding an iso to RHV Manager](/images/iso-add.png "Adding an iso to the storage domain.")

Once it's in place, you should see 'rhel-8.2-x86_64-dvd.iso' available as a cd in 'Boot Options', and can set the First Device in the Boot Sequence to be CD-ROM, and you're off. I had to install an [oVirt SPICE console](https://rizvir.com/articles/ovirt-mac-console/) on my Mac, which let me open the GUI to complete the install by running `remote-viewer ~/Downloads/console.cc`.

One final note is that for some reason, once RHEL 8 was installed, it wouldn't let me register with Subscription Manager - there was an SSL based error, that took some time to bottom out, and I ended up changing `proxy_scheme=http` to `proxy_scheme=https` in the `rhsm.conf` file. Once that was completed, registration went fine and I was back where I was towrds the end of last week!
