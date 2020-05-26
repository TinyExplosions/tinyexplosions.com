---
title: Virtual Insanity
date: '2020-05-07'
tags:
  - homelab
  - tech
  - blog
  - libvirt
  - RHEL 7
---
The future’s made of it, according to [some bloke in a massive hat](https://youtu.be/4JkIs37a2JE), but it’s also a pretty key component to a lab setup, considering a lot of what will be deployed will be build on virtual machines. Yes, I’m aware the docker and containers exist, but a container *platform* works better when deployed on VMs :-) there are, it seems, three main ways to go about the provision of VMs, VMWare, RHEV, and libvirt. 

### VMWare
The defacto standard, and probably the most popular orchestrator of VMs that we see. It’s a fair point to ask why I’d look anywhere else, and if I’m honest, if I was 100% focussed on maximising *customer* based setups, I’d be daft not to look at it. 

The downsides though are that it’s proprietary, fairly expensive (unless you know VMUG types), and I simply would like to at least dabble in the Open Source side -and having employee access to Red Hat Products is really helpful. 

### Red Hat Enterprise Virtualisation 
With VMWare discounted (at least for now, I may end up back there eventually!) the natural contender is our offering, RHEV. I had a look [around the documentation](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.3/html-single/installing_red_hat_virtualization_as_a_standalone_manager_with_local_databases/index) and quickly got scared off by the talk of nodes and manager environments, and just suspect that the learning curve is steeper than I need right now -I’m here to learn container platforms, not virtualisation!

### Libvirt
In many ways the easiest/most basic solution -there is decent support in Cockpit, my server admin tool of choice, and it’s also a good option to build from. Start easy, and if I need to get more complex, I can up my game to one of the other choices. 

Getting going was a matter of installing some packages

```bash
sudo yum install cockpit-machines*
virt-install package
```

Then jumping over to cockpit, and using the GUI. 

I did have to get an OS iso onto BugCity, so I used 

```bash
scp rhel-server-7.8-x86_64-dvd.iso tinyexplosions@bugcity:/var/lib/libvirt/images/rhel-server-7.8-x86_64-dvd.iso
```

to get the iso into `/var/lib/libvirt/images`. Then I created some storage, a bridge network, and spun up my first VM - the GUI is fairly intuitive

[![Cockpit's create VM UI showing configuration of a virtual machine](/images/create-vm.png "UI for creating a VM (ignore the warning on installation source, that's to be sorted another day. Still works as it should though)")](/images/create-vm.png)

It's worth noting that the problems I had with RHEL 8 disappear under Libvirt - all connection to storage is taken care of, so I can run as many RHEL 8 VMs as I could possibly want, so that's a success!

Once you start the VM, you'll see the standard install prompts for RHEL, which you can go through via the console in Cockpit. With that, we have a good base system, and an ability to spin up VMs at will, that bridge their network onto the lan, so we can ssh into them, leaving us one step closer to OpenShift. First though, we need Authentication.