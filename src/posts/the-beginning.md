---
title: The Beginning...
date: '2020-05-05'
tags:
  - homelab
  - tech
  - blog
---
Working with enterprise middleware can be tough sometimes -local development environments can be slow, awkward, or just don’t exist, and AWS prices can get out of hand if you forget to scale down or misconfigure something. The solution, many of my colleagues have found, is in a homelab -a stonking big, expensive, geeky setup running all kinds of acronyms and stuff I don’t understand: “UEFI”, “ovirt”, “SATADOMs” are words? acronyms? brands? I see fly by on the Homelab gChat while I smile and nod. The folk on there though are *very* kind and patient, and have been great with my silly questions so far. 

I had always thought that getting my own setup would be many thousands of pounds, but some new found time in my hands, and some chats led me to [bargainhardware.co.uk](https://bargainhardware.co.uk) -the home of loads of refurbished bits, with really good prices! A lot of playing with various configurator tools later, and for the (not insane) price of a little over £700, I was on the ladder.

Enter `bugcity`, my new foray into the world of homelabs, a Fujitsu Celsius R930n Workstation. My main plan is to run OpenShift, and a couple ancillary bits and bobs, so memory was the most crucial thing I was told. This beastie has 256GB of the stuff, and plenty of unused slots to add more should I need it. It’s also home to a pair of Xeon 8 core processors running at 2.10GHz and about 2TB of storage, on 4 disks (a mix of SSD and spinning platters). 

Yes, I could have probably got more for my money going for a rack server, and it would have totally increased the cool factor, but I don’t (sadly) have a rack, or a garage, or really any place to put a big noisy computer, so hard as is was to resist, this tower is mine. 

The current plan is to have an OpenShift 4 cluster, Ansible Tower, IdM, Gitlab and Quay all running on this, with maybe even a separate Jenkins instance to simulate the sort of setup we see on the job. I have little to no experience of installing and maintaining any of this, so there’ll be lots of stupid questions to my workmates, and I’ll try capture the high and lowlights here for all to laugh at. 

First step (I think) is to install RHEL, and decide what form of VM management I want to use, but that’s for the next post...
