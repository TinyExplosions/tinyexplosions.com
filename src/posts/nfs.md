---
title: NFS, or where to store stuff on your cluster
date: '2020-08-25'
tags:
  - homelab
  - tech
  - blog
  - NFS
  - RHEL 8
---
[My earlier post](/posts/persistent-storage-nfs/) on setting up NFS persistent volumes was based around my Synology, and so is of limited use to people who don't have a similar setup, so given that everything is based of RHEV it seems a good idea to create a VM and run a RHEL-based NFS server. Helps keep everything 'in the box' and is a useful pattern to have.

To begin, I set up a RHEL8 VM, gave it an 80GB partition because I'm being a little frugal, and set it up in my usual manner, adding the usual repos, and registering it with IDM.

```bash
subscription-manager register
subscription-manager attach --pool=<pool id>
subscription-manager repos --disable=*
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
yum update
yum install ipa-client -y
ipa-client-install --enable-dns-updates
```

Once my base system was set up, I ran `yum install nfs-utils`, but to my surprise the package is already installed, so that's a small victory \o/. The next steps are starting the service, setting it to start at boot, and checking the status, all done like so

```bash
systemctl start nfs-server.service
systemctl enable nfs-server.service
systemctl status nfs-server.service
``` 

This now gives us a running nfs server, but it's a bit useless unless we create some shares on it. THis involves creating some suitable folders on the file system, and adding it to the `/nfs/exports/` config file. I'll start with deciding that `/mnt/nfs` is a suitable directory to add my shares, and the first thing I want to store is the OpenShift registry, so I created the folder, added an entry to the exports file (which was empty), and applied it to the system, being sure to set the owner to `nobody`.

```bash
mkdir -p  /mnt/nfs/registry
chown -R nobody: /mnt/nfs/
chmod -R 777 /mnt/nfs/
vi /etc/exports
## Add this to empty exports file
/mnt/nfs/registry       192.168.10.0/24(rw,sync,no_all_squash,root_squash)
## Apply this change
exportfs -arv
```

What's been created is an export that is available to everything on the 192.168.10.0/24 subnet (which is now the subnet of my lab), and allows read and write access, that NFS writes to disk when requested. To check that this has been applied, `exportfs  -s` will list all exports known to the system.

Finally, we want to ensure that the firewall rules allow connections, which can be achieved by the following:

```bash
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --permanent --add-service=nfs
firewall-cmd --reload
```

This should now give us something that OpenShift can connect to, so we need to go back over there to configure a Persistent Volume, and Claim it for the registry. For this we can follow [the previous post](/posts/persistent-storage-nfs/), by visiting Persistent Volumes -> Create Persistent Volume in the console, and adding the following YAML to configure:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 192.168.10.26
    path: /mnt/nfs/registry
```

This will create the volume with the name `registry-pv`, that is pointing to my NFS VM. The reclaim policy is set to `Recycle` so that space is freed up when a claim is deleted. This can also be set to `Retain` or `Delete` if required. In my case, I was keeping it simple so largely left things as default.

Once the Persistent Volume was created, I essentially have 10GB of 'space' that I want to use for the internal image registry, I need to claim sthis volume by creating a Persistent Volume Claim. In a similar manner to PVs, this can be done through the console, by visiting Persistent Volume Claims -> Create Persistent Volume Claim, and adding the following YAML to configure:


```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: slow
```

In order to get the internal image registry to use our new claim, we need to edit the operator, run `oc edit configs.imageregistry.operator.openshift.io`, and under the `spec` section, add the following to make the registry managed, and using the pvc.

```yaml
storage:
  pvc:
    claim: registry-pvc
replicas: 3
managementState: Managed
```

With that, you should be good to go, and if you build and deploy an application, you should see the images in the `/mnt/nfs/registry` directory, and all should be well.