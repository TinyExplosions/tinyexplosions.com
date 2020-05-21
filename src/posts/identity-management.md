---
title: Identity Management
date: '2020-05-09'
tags:
  - homelab
  - tech
  - blog
  - IdM
  - RHEL 8
---
Everything I’m looking to install has built in authentication systems of one sort or another, but why take the easy way, when you can go over the top and mess with a full blown Identity Management solution. 

FreeIPA is a good OSS version, but once again, I decided to go downstream, and attempt to install a [Red Hat Identity Management Server](https://access.redhat.com/products/identity-management). Skimming the documentation led me quickly to the thorny issue of DNS (Integrated or not) and root CA’s (external or integrated), and I started to run out of certainty as to what I needed, and if I could get it suitably configured. 

I’m very much a tinkerer, and so my home network setup is suitably enterprise-y and non trivial, and consider my lack of expertise, a damn miracle that it works at all (I will write it up in further detail at some point I expect). The main thing to note at this point is that I run Ubiquiti everywhere, have the homelab segregated on its own VLAN, and everything talks to the world via a Raspberry Pi running PiHole, that itself is configured to use DNS over HTTPs. It all means I probably could set up just about any type of setup, as long as I could figure out what the setup needs to be. 

Never one to let the unknown get to me, I spun up a RHEL 8 VM, and did the usual subscription manager dance, and ensured I had the correct repos enabled

```bash
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
```

After a few hours of things not working, and getting really frustrated with it all, I reached out to some of my wonderful colleagues, who helped me pinpoint that I was looking at the wrong feckin docs! Yes, I had RTFM, just it was the wrong FM! Finally, I [opened the correct documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/installing_identity_management/index) and set about following the instructions to configure an IdM Server with integrated DNS, and an integrated CA as root CA. That step was ok, just needed to enable correct repositories, install the required modules, and then start the installer

```bash
yum module enable idm:DL1
yum distro-sync
// Install With Integrated DNS
yum module install idm:DL1/dns
ipa-server-install
```

The installer asks a load of questions, and I (not entirely blindly) accepted mostly the defaults, as it was reading the correct stuff from my `/etc/hosts` file etc, but then I was temporarily stumped when the installer failed with some name server based errors, but good old google came to the rescue, and so I re-ran the installer with the correct flag:

```bash
ipa-server-install --allow-zone-overlap
```

This time, the installer completed successfully, and going to idm.bugcity.tech in a browser gave me a lovely IdM GUI to log into (as I’ve said before, I love a good GUI). 

![Red Hat Identity Manager Interface](/images/idm.png "IdM Dashboard - lots of stuff here to learn.")

There will be a lot to dig into I’m sure to configure this properly, as well as getting it to play well with Tower, OpenShift etc etc, but as per usual, it all looks to have installed ok, and so all that is for another day.