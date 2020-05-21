---
title: OpenShift and LDAP
date: '2020-05-21'
tags:
  - homelab
  - tech
  - blog
  - OpenShift
  - LDAP
  - IdM
---
Now that I *think* I have [the main woes of my OpenShift cluster](/posts/it-wasnt-dns) sorted, it's time to turn my attention to some other things, and the first one to investigate is configuring the cluster to use my IdM setup to provide LDAP authentication.

Configuration comes in two parts - Authentication, and Authorisation/Role Based Access Control (RBAC). The first is the most basic connection, have you supplied the correct credentials, does the user belong in a specific group, or set of groups, that sort of thing. RBAC is more involved, and creates a periodic job that syncronises groups and pulls them into OpenShift, so you can apply specific Roles and Policies to them. This is the part were you could define that group `foo` in LDAP has the role `cluster-admin` etc etc. I leaned heavily [on this article](http://v1.uncontained.io/playbooks/installation/ldap_integration.html) for most of what went on below, so it's worth having a scan over first.

### LDAP Configuration

To begin, it was into IdM to get that side of things squared away. I created a group, `ocpusers`, and already have a `superusers` group that I am using as a 'catch all' one to give admin or whatver the highest power privileges are on any connected system, so this should be a good baseline - membership of either group will allow one to log into the cluster, and someone in the `superusers` group will be a cluser administrator. More complex setups can be considered and added later, but this is a good baseline from which to start. I also have two users to test with, `tinyexplosions` who is in both the `ocpusers` and `superusers` groups (among others), and `ocp_user` who is only in the `ocpusers` group.

It was then time to fire up `ldapsearch` to check out how my particular configuration reports things. Lets get the user `tinyexplosions`. This will give us the dn's of the various items we will want to query down the road.

```bash
ldapsearch -x  -LLL -H ldap://idm.bugcity.tech:389 \ 
-D "uid=admin,cn=users,cn=compat,dc=bugcity,dc=tech" \ 
-w <password> -b "cn=users,cn=accounts,dc=bugcity,dc=tech" \ 
-s sub "uid=tinyexplosions"

dn: uid=tinyexplosions,cn=users,cn=accounts,dc=bugcity,dc=tech
givenName: Tiny
sn: Explosions
uid: tinyexplosions
cn: Tiny Explosions
displayName: Tiny Explosions
initials: TE
gecos: Tiny Explosions
krbPrincipalName: tinyexplosions@BUGCITY.TECH
<snip... />
mail: tinyexplosions@bugcity.tech
memberOf: cn=superusers,cn=groups,cn=accounts,dc=bugcity,dc=tech
memberOf: cn=ocpusers,cn=groups,cn=accounts,dc=bugcity,dc=tech

```

### OCP Configuration

Then, it was into OpenShift, to the cluster OAuth configurater `/k8s/cluster/config.openshift.io~v1~OAuth/cluster`, and then I used the UI to add an LDAP Identity provider. Based on the info gleaned from above, filled in the fields as follows:
* Name: ldap
* URL: ldap://idm.bugcity.tech:389?uid
* BindDN: uid=admin,cn=users,cn=compat,dc=bugcity,dc=tech
* Bind Password: &lt;pw&gt;
* ID: dn
* Preferred Username: uid
* Name: cn
* Email: mail
* CA File: &lt;upload ca taken from IdM&gt;


Once that was filled in, I waited a couple of minutes for it to apply, then logged out of the cluster, and was greeted with a new login screen, which was promising

![OpenShift login screen showing kube:admin and ldap options](/images/ocp-login.png "OpenShift login screen, with a new, shiny 'ldap' button!")

Using the new ldap functionality, I input the details for the user `tinyexplosions`, and it logged me in! Logging out, and retrying authentication with the `ocp_user` user also allowed me to log in, which was expected, but not desired. From this point it was time to dig into the yaml, and make some changes in order to restrict auth by group.

### Group Restrictions

The trick to narrow down the authentication to specific groups is to get the ldap url correct. If we look at the existing url, `ldap://idm.bugcity.tech:389?uid`. we can see that we're looking for a valid `uid` to be returned. We also want to see if a user is in a specific group. This is done by appending `(memberOf=<group>)` declarations to the url, so in our case we want to check for `ocpusers` group, so we can append `(memberOf=cn=ocpusers,cn=groups,cn=accounts,dc=bugcity,dc=tech)`. The LDAP standard also allows for the differing operators to be added to join or exclude multiple entries, for example `(&(memberOf=x)(memberOf=x))` would check that a user is a member of both `x` and `y` groups.

For our simple example, membership of either pertinent group is just fine, so we want to add `(|(memberOf=cn=ocpusers,cn=groups,cn=accounts,dc=bugcity,dc=tech)(memberOf=cn=superusers,cn=groups,cn=accounts,dc=bugcity,dc=tech))` to the url. It is appended to our url, with `??` coming before it, and so the final yaml for my solution looks like below:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  annotations:
    release.openshift.io/create-only: 'true'
  creationTimestamp: '2020-05-18T21:41:18Z'
  generation: 14
  name: cluster
  resourceVersion: '2428122'
  selfLink: /apis/config.openshift.io/v1/oauths/cluster
  uid: 91f17651-559f-4f9b-b5fe-300f64e3e48a
spec:
  identityProviders:
    - ldap:
        attributes:
          email:
            - mail
          id:
            - dn
          name:
            - cn
          preferredUsername:
            - uid
        bindDN: 'uid=admin,cn=users,cn=compat,dc=bugcity,dc=tech'
        bindPassword:
          name: ldap-bind-password-n8t8w
        ca:
          name: ldap-ca-6mqjz
        insecure: false
        url: >-
          ldap://idm.bugcity.tech:389/cn=users,cn=accounts,dc=bugcity,dc=tech?uid??
          (|(memberOf=cn=ocpusers,cn=groups,cn=accounts,dc=bugcity,dc=tech)
          (memberOf=cn=superusers,cn=groups,cn=accounts,dc=bugcity,dc=tech))
      mappingMethod: claim
      name: ldap
      type: LDAP
```

Applying the above, and waiting the appropriate time for it to apply, and it was back to the login screen to test. First up I was able to successfully log in with both accounts. From there, it was back to IdM, and I removed user `ocp_user` from the `ocpusers` group, and was then unable to log into OpenShift with them. The user `tinyexplosions` continued to work, and remained able to log in with membership of *either* `superusers` or `ocpusers` enabled, or with membership of both groups enabled, which is what I wanted.
