---
title: Tower LDAP Integration
date: '2020-05-14'
tags:
  - homelab
  - tech
  - blog
  - Ansible
  - Tower
  - IdM
  - LDAP
---
With the OpenShift install taking longer than expected, and with it having some... issues... I decided to knock it on the head for a bit, and refocus on Ansible, and specifically integrating it with IdM -well, there’s no point in having a central identity provider, if I’m not going to use it to centrally identify people. 

IdM comes with LDAP built right in, and [the official documentation is pretty good](https://docs.ansible.com/ansible-tower/latest/html/administration/ldap_auth.html), as is a [slightly older QuickStart](https://www.ansible.com/blog/getting-started-ldap-authentication-in-ansible-tower) I was pointed to, but there were a couple of things I had to figure out, so it’s worth sharing. 

First off, install `ldapsearch` -it’s invaluable in verifying the correct values for each field, because as good as IdM is, it doesn’t seem to have a central area where you can see Distinguished Names, Common Names etc, and so the command line tool is darn useful. 

Second (and I wish I’d actually bothered to do this earlier), familiarise yourself with the location of Tower’s log files `/var/log/tower/tower.log` is the one you want, and the only place you’ll be able to see any errors in your config -the UI will only say “username or password incorrect”, which is less than helpful. 

With that preamble out the way, I next created a user and a group in IdM (‘tower_admin’ and ‘tower_administrators’
Respectively)  to test the setup out, then fired up the ldap settings screen in Tower. 

A combination of the linked documentation and playing with the command line led me to the correct LDAP URI, and BIND DN, and gave me the basis to run all queries on.

```bash
ldapsearch -x  -H ldap://idm.bugcity.tech:389 -D "uid=admin,cn=users,cn=compat,dc=bugcity,dc=tech" -w <password>
// get the tower_admin user
ldapsearch -x  -H ldap://idm.bugcity.tech:389 -D "uid=admin,cn=users,cn=compat,dc=bugcity,dc=tech" -w <password> -b "cn=users,cn=accounts,dc=bugcity,dc=tech" "(uid=tower_admin)"
# tower_admin, users, accounts, bugcity.tech
dn: uid=tower_admin,cn=users,cn=accounts,dc=bugcity,dc=tech
givenName: Tower
sn: Administrator
uid: tower_admin
cn: Tower Administrator
displayName: Tower Administrator
initials: TA
gecos: Tower Administrator
krbPrincipalName: tower_admin@BUGCITY.TECH
objectClass: top
objectClass: person
objectClass: organizationalperson
objectClass: inetorgperson
objectClass: inetuser
objectClass: posixaccount
objectClass: krbprincipalaux
objectClass: krbticketpolicyaux
objectClass: ipaobject
objectClass: ipasshuser
objectClass: ipaSshGroupOfPubKeys
objectClass: mepOriginEntry
loginShell: /bin/sh
homeDirectory: /home/tower_admin
mail: tower_admin@bugcity.tech
krbCanonicalName: tower_admin@BUGCITY.TECH
ipaUniqueID: dcdf8396-9421-11ea-816b-566f23430002
uidNumber: 1145600001
gidNumber: 1145600001
mepManagedEntry: cn=tower_admin,cn=groups,cn=accounts,dc=bugcity,dc=tech
memberOf: cn=ipausers,cn=groups,cn=accounts,dc=bugcity,dc=tech
memberOf: cn=tower_administrators,cn=groups,cn=accounts,dc=bugcity,dc=tech
krbLastPwdChange: 20200513131525Z
krbPasswordExpiration: 20200513131525Z
krbLoginFailedCount: 0
krbExtraData:: AALt8rtecm9vdC9hZG1pbkBCVUdDSVRZLlRFQ0gA
```

This user gave me most of the information needed to fill in the rest of the fields, such as the attribues to map, the DN of the group, and the attribute used to search for the user (`uid`).

![Tower LDAP integration screen](/images/ldap-settings.png "Ansible Tower LDAP configuration showing User and Group search.")

With all the config in place, I still couldn’t get it working, and so I verified and changed every setting what felt like hundreds of times, before thinking to look in Tower’s logs. Opening the log file revealed

```bash
2020-05-13 15:54:45,497 WARNING  django_auth_ldap Caught LDAPError while authenticating tower_admin: CONNECT_ERROR({'desc': 'Connect error', 'info': 'error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed (self signed certificate in certificate chain)'},)
```

(Yes, I know I need to sort out a load of certificate stuff, it’s on the long finger -mostly because I’d really like to have IdM use LetsEncrypt as a CA, but I don’t know where to start that journey). Flipping the 'LDAP start tls' toggle to off in Tower led to a successful login, and all seemed well. 

To test the functionality a little more, there were some extra steps I wanted to verify. To ensure I’m parsing group membership correctly, it was back to IdM to create what will be my ‘main’ user, ‘tinyexplosions’, and expressly not add it to Tower Administrators. Trying to log into tower was a no go, so that box has been checked. 

Secondly, I wanted to return to the `is_superuser` clause in the tower config. Even though I’m the only user here, it’s worth fleshing this out, so I wanted to look at adding it. In IdM, I added a new group, calling it 'super_users', and added the 'tinyexplosions' user to both this and the 'tower_administrators' groups. In Tower, I added the following in the 'LDAP user flags by group' field

```bash
{
 "is_superuser": [
  "cn=super_users,cn=groups,cn=accounts,dc=bugcity,dc=tech"
 ]
}
```

This means that an LDAP user who is a member of the above group will be a super user in tower, giving the full access. This is verified by logging in as both 'tinyexplosions' and 'tower_admin' and seeing the differences


[![Tower LDAP user comparison. One user can see all options, the other cannot](/images/tower-superuser.png "User 'tinyexplosions' is a Tower Administrator, user 'tower_admin' is not.")](/images/tower-superuser.png)

This should be a good baseline for going forward, as I have a superuser group to set roles in an other applications I want to integrate with LDAP, as well as having Tower correctly integrated.

### Addendum

I was pointed out to me that the easiest way to have my IdM certs trusted was to enroll the machine in IdM, and that that would even allow me to log into the machine over LDAP. Queue lightbulb moment, and feeling a bit daft - of course that makes sense, why use any other method to log in!

```bash
yum install ipa-client
ipa-client-install --enable-dns-updates
```

Was all it took (adding in my IdM server details), and after a reboot, I could enable the 'LDAP start tls' toggle, and feel even more secure in my authentication.

There was a little bit of mucking around in IdM to get thingd as I like, namely
* Added a new group, `idm_client_sudoers`, which will govern who can auth into hosts
* Added a Host Group, `idmclients` and added the new client `tower.bugcity.tech` to it
* Added HBAC rule called `idmclient` and added `idm_client_sudoers` user group, and the `idmclients` host group to it
* Added Sudo rule `sudoers` and added `idm_client_sudoers` to it

I'm already starting to see how groups in LDAP can get out of control, but in for a penny, in for a pound!

