---
title: Cron Jobs in OpenShift
date: '2020-05-22'
tags:
  - homelab
  - tech
  - blog
  - OpenShift
  - LDAP
  - IdM
---
Yesterday [was a successful day](/posts/ldap-on-openshift), and on the face of it - today should have been child's play. After all, I have created a sync job for OpenShift, I can run it just fine from the command line, all that needs to be done is run it every x mins/hours/days whatever. Heck, there's even a section in the OpenShift Console that's labelled 'Cron Jobs' - I'll be done in minutes.

[![OpenShift Console with 'Cron Jobs' highlighted](/images/CRON-JOBS.png "Look, it's a section labelled 'Cron Jobs' - this is going to be a piece of cake...")](/images/CRON-JOBS.png)

It was then that my naivety with the workings of OpenShift came to the fore, leading to a bit more work and googling that I'd expected, but I guess that's why I'm here - to go through the hassle so you don't have to. Or future me doesn't have too when I want to add another cron job in a few months.

Immediately, it was not apparant how to add a new cron job, given that I already had a working yaml file - so it was off to google, and [this template](https://github.com/redhat-cop/openshift-management/blob/master/jobs/cronjob-ldap-group-sync.yml) popped up as a likely candidate. Scanning through it, the actual job looked pretty similar to mine, so this was going to be the path to success. Template downloaded, it was a matter of running

```bash
oc create -f ./cronjob-ldap-group-sync.yml -n openshift-config
```

to get it into OpenShift, in the `openshift-config` project. Why that one? Mostly because that was where all the LDAP config for authentication went. I'd probably suggest creating a new project for this, but live your own life!

This now had the template added to OpenShift. Now it was time to create something with it.

For this it was into the 'Developer' menu, then selected project `openshift-config` from the dropdown at the top of the page, then clicked the 'From Catalog' box (after pausing a moment to lament the lack of internationalisation for OpenShift. I want my catalogue damnit). Sticking `cronjob-ldap-group-sync` in the search box brought up my template, and 'Instantiate Template' got me filling in some variables.

One of the first things I noticed was a variable named 'Service Account' - oops, guess I'd better create one of those - quick jump over to the Users area in another browser, and soon `ldap-group-syncer` was born, and given appropriate permissions on the account.

Most of the other variable were straightforward, as I was copying values from my own earlier work, but there were a couple that gave me some pause. The default value for 'Group Filter' (`(&(objectclass=ipausergroup)(memberOf=cn=openshift-users,cn=groups,cn=accounts,dc=myorg,dc=example,dc=com))`) is a lot more restrictive, and therefore better than mine, and so I might investigate something similar later, but for now I stuck with the tried and tested `(objectClass=groupOfNames)`. I also left 'Image' untouched at `registry.access.redhat.com/openshift3/ose-cli` even though there's probably an OpenShift4 version, but if it works it works. 

Finally, there was the manner of the `LDAP sync whitelist` input field. Yup, not a text area, an input field. Clearly I had two lines to add (as mentioned in previous post), but how can I add a line break in an input field? The answer is that you try a bit, then stick a space in, click 'Create' and pretend that it's all grand. Back away slowly from the keyboard and have a snack...

Time passed, video calls were had, and then I flipped back over to the OpenShift console, to see a few red "!" - that's not good, lets see what's going on. As I kinda suspected, my whitelist was causing problems. I needed to get the dn's for the whitelist groups on to separate lines. To do this, I went to 'Config Maps' in the sidebar, ensured the correct project (`openshift-config`) was selected, and selected `ldap-config`. Into the YAML, and changed whitelist to

```yaml
whitelist.txt: |-
    cn=superusers,cn=groups,cn=accounts,dc=bugcity,dc=tech
    cn=ocpusers,cn=groups,cn=accounts,dc=bugcity,dc=tech
```

click on save, and in theory I was done. But would have to wait an hour for the job to be run again. I can't have that, so over to the aforementioned, but heretofore unclicked 'Cron Jobs' item, selected my job, and jumped into the YAML to set schedule to `*/1 * * * *` (that's 'every minute' for those that, like me, don't read cron), and within 60 seconds, my job was running again, and this time - success!

[![Successful run of my cron job](/images/complete.png "Job's a good 'un")](/images/complete.png)

So, there we have it. OpenShift is talking to LDAP on IdM to perform Authentication, and every hour will sync LDAP groups with OpenShift, so we have RBAC in place. At some point, it'll be good to add a 'prune' job to run as well, but given that it's only me around, and I'm not going to be adding many users, it's at a lower priority.

What's the next step you ask (quietly)? Well, either some Ansible based Tower fun, or some Service Mesh shenanigans - you'll have to check back to find out.
