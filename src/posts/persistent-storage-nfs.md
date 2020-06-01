---
title: Persistent Storage (NFS flavour)
date: '2020-06-01'
tags:
  - homelab
  - tech
  - blog
  - Synology
  - PV
  - OpenShift
  - NFS
---
After our little sojourn into app dev yesterday, it's dropping back to infrastructure today to talk Persistent Volumes (PV's) and Persistent Volume Claims (PVC's). For the uninitiated, a PV is like your big disk, and each PVC claims a certain part of that, kind of like a folder. Wile all the kids these days love talking about 'ephemeral' and 'stateless' apps, having some amount of persistent storage is useful for a variety of usecases - not least of which is for an Image Registry.

So, it was time for PV's, but what which one to choose? Given that my cluster is running on RHV, I could spin up a drive or some space on there, but that seemed a little complicated in many ways, so my eyes turned to my NAS... well, it does have 'Storage' as part of it's acronym... I have a Synology DS218+ to hold media, but it has plenty of spare space on it, so lets get adding an NFS share to it.

[The Synology documentation](https://www.synology.com/en-us/knowledgebase/DSM/tutorial/File_Sharing/How_to_access_files_on_Synology_NAS_within_the_local_network_NFS) gives a really good overview, and following that I carved out a 50GB share, tied to the subnet of my lab. There are many other authentication options that can be used, but for now I'm going with simple (complexity can be added later.)

[![Synology NFS configuration screen](/images/nfs-creation.png)](/images/nfs-creation.png)

Once this was in place, it was over to OpenShift to create Persistent Volume. This can be done through the ui, by visiting Persistent Volumes -> Create Persistent Volume in the console, and adding the following YAML to configure:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: synology
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  nfs:
    server: 192.168.0.200
    path: /volume1/Homelab
```

This will create the volume with the name `synology`, that is pointing to my NFS mount point (`/volume1/Homelab on 192.168.0.200`). The reclaim policy is set to `Recycle` so that space is freed up when a claim is deleted. This can also be set to `Retain` or `Delete` if required. In my case, I was keeping it simple so largely left things as default.

Once the Persistent Volume was created, I essentially have 50GB of 'space' that I can use to store things. If I want to use any of that for an application, or for the internal image registry, I need to claim some of that volume. That is done by creating a Persistent Volume Claim. In a similar manner to PVs, this can be done through the console, by visiting Persistent Volume Claims -> Create Persistent Volume Claim, and adding the following YAML to configure:


```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: synology-pvc-images
  namespace: psychic-octopus
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: slow
```

This is creating a claim called `synology-pcvc-images` and I'm specifying that it is 5GB in size. I also add `storageClassName: slow` to help reference it back to the PV created before. Saving the YAML file and waiting a few seconds should leave you with a result similar to below, with my PVC Bound to the correct PV.

[![OpenShift PVC screen showing correctly bound Claim](/images/pvc-success.png)](/images/pvc-success.png)

You might notice in the screenshot above that it is a little misleading. Or at least, it was to me before checking with a colleague. The 'Capacity 50Gi' looked very wrong - that was the entirety of the PV, but I had only specified 5Gi in my configuration. It would seem that this little nugget refers to the face that it can expand *up to* 50Gi if it needs to, but verifying the YAML for the claim shows that 5Gi is correctly specified as it's size.

So there you have it - my cluster is now able to save stuff on my NAS - maybe the next step is hooking the claim up to the image registry... look out for that in a future installment.
