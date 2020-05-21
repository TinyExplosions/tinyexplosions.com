---
title: It's Always DNS
date: '2020-05-21'
tags:
  - homelab
  - tech
  - blog
  - OpenShift
  - LDAP
  - IdM
---


ldapsearch -x  -H ldap://idm.bugcity.tech:389 -D "uid=admin,cn=users,cn=compat,dc=bugcity,dc=tech" -w Fmjogc8206 -b "cn=accounts,dc=bugcity,dc=tech" "(objectClass=groupOfNames)"