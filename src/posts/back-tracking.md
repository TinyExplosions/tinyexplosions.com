---
title: Back Tracking
date: '2020-05-10'
tags:
  - homelab
  - tech
  - blog
---
I’m writing up all of this homelab work in real time (or maybe off by only a couple of days), so I suspect this may become the first in a series of reconsiderations and mind changes. I’ve often said that effective Consultants and Architects have strong opinions that are loosely held (I don’t necessarily agree that it also tracks for VCs or Product Engineers or other types, but in professional services it definitely has legs). It’s good to have a firm idea, and to be able to convey it to a customer, but also not to dwell on it too much if it’s rejected or ignored. 

In this instance, it was a couple of remarks from a colleague about IPA being available in OpenShift 4.4 for RHV, and [this article from our docs](https://docs.openshift.com/container-platform/4.4/installing/installing_rhv/installing-rhv-default.html#installing-rhv-requirements_installing-rhv-default) that has led to me moving away from libvirt for managing my VMs, and going with Red Hat Virtualisation -a lot of the built in orchestration with the installers etc is there, and Mainly the existence of an IPI route to OpenShift swings the balance for me. 

It’s not a bad time to do it either, as I only have a single VM running IdM at the moment, and even though I should be able to move it easily under RHV, in a worst case scenario, it’s an easy item to recreate (as I now know what docs to look at, and which config to use)

So, libvirt out, RHV in, and let’s see what other fundamental changes get made over the next little while as I learn more about how everything hangs together.