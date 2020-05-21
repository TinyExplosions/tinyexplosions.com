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

### Syncing Groups

This is where things get a little trickier. [The official documents](https://docs.openshift.com/container-platform/4.4/authentication/ldap-syncing.html) are pretty good, so I sugegst reading through them first. Then fire up a text editor and write some more YAML (christ, does OpenShift love a bit of YAML). Nothing too fancy here though:

```yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://idm.bugcity.tech:389
insecure: false
ca: "<relative/link/to/cert.pem>"
bindDN: "uid=admin,cn=users,cn=compat,dc=bugcity,dc=tech"
bindPassword: "<password>"
groupUIDNameMapping:
    "cn=superusers,cn=groups,cn=accounts,dc=bugcity,dc=tech": openshift_admins
rfc2307:
    groupsQuery:
        baseDN: "cn=accounts,dc=bugcity,dc=tech"
        scope: sub
        derefAliases: never
        filter: (objectClass=groupOfNames)
        pageSize: 0
    groupUIDAttribute: dn
    groupNameAttributes: [ cn ]
    groupMembershipAttributes: [ member ]
    usersQuery:
        baseDN: "cn=users,cn=accounts,dc=bugcity,dc=tech"
        scope: sub
        derefAliases: never
        pageSize: 0
    userUIDAttribute: dn
    userNameAttributes: [ uid ]
```

Of interest are the `baseDn`, `filter` and attribute fields. All of these should be familiar though. The `baseDn` is the one secified in the Tower integration, so you can copy those from the LDAP User Search and LDAP Group Search fields we added [in our configuration](/posts/tower-ldap-integration) last week. Also, the `filter` attribute is taken from the Tower config too - it's the LDAP Group Search filter, so can be added here too. The various attribute fields are to get the full dn of users, and the attribute we want to appear as a username in OpenShift. Finally, we have `groupUIDNameMapping`. This allows us to have a group with one name in LDAP, and another in OpenShift. In this case, we take our `superusers` group in LDAP, and call it `openshift_admins` in OCP.

As is stands, running this will take every group LDAP sees and add them as groups in OpenShift. Clearly this isn't desirable, and so that is where whitelists and blacklists come in. They are files that you can use to explicitly include or exclude groups from the sync. In our example, we only want the `superusers` and `ocpusers` groups to sync their info, so we add their dn's to a whitelist file

```text
cn=superusers,cn=groups,cn=accounts,dc=bugcity,dc=tech
cn=ocpusers,cn=groups,cn=accounts,dc=bugcity,dc=tech
```

Once these resources are in place, you can run it against OpenShift (leave out the `--confirm` if you just want to test the output.

```
 oc adm groups sync --sync-config=./usersync.yaml --whitelist=./whitelist.txt --confirm
```

During some of my runs, I saw the following error, all it meant was the group `openshift_admins` already existed (and wasn't created by an earlier sync), and so I deleted the group and ran it again.

```bash
group "openshift_admins": openshift.io/ldap.host label did not match sync host: wanted idm.bugcity.tech, got
```

After a successful run, I could see my users and groups in OpenShift, and the only thing left to do was make the `openshift_admins` group cluster adminsitrators, meaning `tinyexplosions` can mess up anything they want!

```
oc adm policy add-cluster-role-to-group cluster-admin openshift_admins
```

Repeating the logout/login dance with user `tinyexplosions` and I was greeted with the Administrator overview, and some lovely errors to look into - but from an auth point of view, it was a rousing success.


![OpenShift admin dashboard with user TinyExplosions authenticated](/images/ldap-user-admin.png "TinyExplosions logged in as a cluster admin (ignore the errors, that'll get sorted later)")

Another useful command to know is `oc adm groups prune --sync-config=./usersync.yaml --whitelist=./whitelist --confirm` - this will remove users who no longer exist in the groups and keep you in tip top shape. 

The only thing left to do now is make all the above a cron job, so that we can periodically sync, but that is for next time.