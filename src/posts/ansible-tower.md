---
title: Ansible & Tower
date: '2020-05-12'
tags:
  - homelab
  - tech
  - blog
  - Ansible
  - Tower
  - RHEL 8
---
Real work was the priority today, so not a huge amount of playtime on the lab, but I did get Ansible and Tower configured (at least in it's most basic incarnation). The Ansible install followed the [official documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-rhel-centos-or-fedora), with the enabling of the correct repo, and a `yum install`

```bash
subscription-manager repos --enable ansible-2.9-for-rhel-8-x86_64-rpms
yum install ansible
```

After that was a quick check of the minimum specs for tower, and the requisite bumping of resources (4 GB RAM, 2 CPUs) and a VM restart, then to the [Ansible Tower documents](https://docs.ansible.com/ansible-tower/latest/html/quickinstall/download_tower.html) to figure stuff out. It was a download of the bundled Installation Program, then modify the default inventory to add some passwords

```yaml
[tower]
localhost ansible_connection=local

[database]

[all:vars]
admin_password='password'

pg_host=''
pg_port=''

pg_database='awx'
pg_username='awx'
pg_password='password'

rabbitmq_port=5672
rabbitmq_vhost=tower
rabbitmq_username=tower
rabbitmq_password='password'
rabbitmq_cookie=rabbitmqcookie

# Needs to be true for fqdns and ip addresses
rabbitmq_use_long_name=false
# Needs to remain false if you are using localhost
```

And a `./setup.sh` to get going. First time round there was an error thanks to `rsync` not being installed, but once I did that we had Tower up and running in a jiffy!

![Red Hat Tower Interface](/images/tower-dashboard.png "Ansible Tower Dashboard - a bit empty for now, but will soon get filled with playbooks.")

The plan was to get LDAP auth setup between Tower and Idm, but the siren song of [OpenShift IPI on RHEV](https://docs.openshift.com/container-platform/4.4/installing/installing_rhv/installing-rhv-default.html) was too strong, and I got a little distracted by that, but more on that another day.