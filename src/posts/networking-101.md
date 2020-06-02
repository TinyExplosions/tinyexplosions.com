---
title: Networking 101
date: '2020-05-26'
tags:
  - homelab
  - tech
  - blog
  - Ubiquiti
  - IdM
  - PiHole
  - Dnsmasq
---

My networking setup is (quite probably needlessly) quite complicated. As I happen to live in [the first city to get a full fibre roll-out](https://www.cityfibre.com/gigabit-cities/milton-keynes/?utm_campaign=crowdfire&utm_content=crowdfire&utm_medium=social&utm_source=pinterest) (even though it's not even a city), last year saw my road dug up (twice!) and lovely big green boxes appear here and there. A couple months later, and I was signed up for a 500Mbps symmetrical line in. Given that I was getting new internet, it would be rude not to re-evaluate my entire network setup, and so thanks to a kind donation of a couple of [AP-AC-LR](https://www.ui.com/unifi/unifi-ap-ac-lr/) access points, an afternoon of ladder borrowing, drilling and external cat-6 running, and some reward points from work that could be turned into Amazon vouchers, and thus shiny toys, I was basically all in on Ubiquiti's line of products, and had a hard line from the front of the house to my office.

In the end I picked up a [USG security gateway](https://www.ui.com/unifi-routing/usg/), 2 [US‑8‑60W 8 port hubs](https://www.ui.com/unifi-switching/unifi-switch-8/), and a [UAP-AC-IW In-Wall access point](https://inwall.ui.com). I already had a Synology NAS, and so Docker on that was put into service to run the Controller software to save me getting a Cloud Key. Finally, because I'm a glutton for punishment, a [Raspberry Pi 4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) was also aquired, to run [Pi Hole](https://pi-hole.net).

[![Screenshot showing the network devices connected together.](/images/network-device-setup.png "The physical layout of my network devices")](/images/network-device-setup.png)

Once everything arrived, was plugged in, and recognised by the Controller software (which took longer than it should have due to some dodgy RJ45 sockets and my less than stellar crimping skills), it was time to sort out some networks. Eventually I settled on three:

* `Computers n Shit` -this is the 'main' computers in the house - our laptops, phones, the Nas, the Raspbrry Pi etc (192.168.0.0/24)
* `Dangerous home assistant yokes` -all the IoT devices we have, as well as game consoles, TV, DVR, etc (192.168.42.0/24)
* `Homelab` -the homelab -currently just BugCity (178.0.0.0/24)

The way Unifi works is that all of these networks have a gateway IP of the &#42;.1 of their subnet (192.168.0.1, 192.168.42.1, 178.0.0.1), but this isn't an actual device, it's all internal. There is also a WAN network defined, that has the PPPoE connection to my ISP, and *this* is where I define the Raspberry Pi (on 192.168.0.210) as the DNS server for everything that connects to any of the networks.

There are also suitable firewall rules in place so that `Computers n Shit` can talk to anywhere, `Dangerous home assistant yokes` is isolated on its own, with a couple of devices specified by MAC address as being able to connect to the NAS, and `Homelab` is totally on its own.

Finally, I have 2 wireless networks defined, `HTTP418` (for the Coffee Pot Control Protocol fans) which is part of `Computers n Shit`, and `dodgy-bois` which is for `Dangerous home assistant yokes`. Unifi also lets me choose networks for specific ports on the switches, so that is also used to specify the `Homelab` and some `Dangerous home assistant yokes` clients.

So far, so (fairly) straightforward (or maybe a little overkill). It's in the Raspberry Pi though, where things get interesting.

### Pi Hole
Pi Hole is [a black hole for Internet avertisements](https://pi-hole.net/) - basically, it blocks known advert urls at a DNS level. This gives one ad blocking before it even hits your network, saving you bandwidth, and annoying spouses, as they just *love* clicking on the first result in google for any search, which is always an ad.

The install and basic setup is pretty well covered in the documentation, and I was soon up and running, but it was then I started to want to get a little... esoteric. I'd read a couple of articles on DNS-over-HTTPS, and it seemed like a good idea to me - a fun way to stop any ISP shenanigans, and heck if [the UK Government was against it being turned on by default in Firefox](https://www.theregister.co.uk/2019/09/24/mozilla_backtracks_doh_for_uk_users/), it *must* be a good idea! Once again, [the documentation](https://docs.pi-hole.net/guides/dns-over-https/) was spot on, and soon I was up and running and ready to go. Or was I?

### Internal Resolution
Once I had the basic setup working, it was time to take it one step further. I have a domain (lets call it `foo.com`) that points to my static IP, and is port forwarded to my NAS. This is massively convienient, and allows me to stream content when I'm away for work etc, and not have to remember an IP address. However, if I was inside the house, I'd prefer `foo.com` to resolve to the local address, to avoid a hop outside of my network. Also, it would be really handy to have all my lab DNS handled by IdM. I am using it for LDAP on the lab, and it makes sense to use it as a DNS resolver too. Keep everything in one place and all that.

Thankfully, Pi Hole ships a variant of `dnsmasq` called FTLDNS. This allows me to use standard `dnsmasq` declarations. I created a file, `/etc/dnsmasq.d/bugcity.conf` and entered some config, as follows:

```bash
localise-queries

no-resolv

domain-needed
bogus-priv

expand-hosts
domain=foo.com
local=/foo.com/
server=/0.0.178.in-addr.arpa/178.0.0.17
server=/bugcity.tech/178.0.0.17
```

* `localise-queries` means that it will look in `/etc/hosts` for entries
* `no-resolv` means that it will ignore `/etc/resolv.conf`
* `domain-needed` means it won't forward A or AAAA queries for plain names, without dots or domain parts. If the name is not known from `/etc/hosts` or DHCP then a "not found" answer is returned
* `bogus-priv` means that all reverse lookups for private IP ranges (ie 192.168.x.x, etc) which are not found in `/etc/hosts` or the DHCP leases file are answered with "no such domain"
* `expand-hosts` means that the domain will be added to simple names (without a period) in `/etc/hosts` in the same way as for DHCP-derived names. 
* `domain` and `local` let me expand simple names and add `foo.com`, and tells FTL that it's locally resolved, so never go out to DNS for any `foo.com` addresses (the NAS address is specified in `/etc/hosts`)
* `server` these entries perform forward and reverse lookup from this device to the IDM install that has the IP 178.0.0.17. Basically, this means that any requests for a `bugcity.tech` address will be kicked to my IdM install, and handled there.

In IdM, I have a few things configured as pictured. Some come automatically when a machine is enrolled with IdM, `api.ocp` and `apps.ocp` are the IPs of my OpenShift cluster.

[![Red Hat Identity Manager DNS page](/images/Idm-DNS.png "IdM DNS Zone for bugcity.tech.")](/images/Idm-DNS.png)

So, there we have it. Some decent routing for requests so that the things I want to keep internal stay internal, and anything that needs to go outside the network are using https, so no snooping by Vodafone :)
