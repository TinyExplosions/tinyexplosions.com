---
title: Infra Nodes in OpenShift
date: '2020-06-08'
tags:
  - homelab
  - tech
  - blog
  - RHEV
  - OpenShift
---
Installer Provisioned Infrastructure (IPI) is undoubtedly a great way to install OpenShift. A lot of sensible defaults have been made by Red Hat, and when it completes, you get a nice cluster, with 3 master, and 3 worker nodes.

Infrastructure nodes were a clear concept in the days of OpenShift 3, the Control Plane was clearly split into Master and Infra nodes, and then your App nodes held all your, well, Applications. If you look at the documentation for OCP 4, you'll see that Infra nodes barely get a mention. We simply have masters and workers, and so if you inspect your nodes on a fresh install, you get:

```bash
$ oc get nodes
NAME                       STATUS   ROLES    AGE     VERSION
ocp-jb9nq-master-0         Ready    master   20d     v1.17.1
ocp-jb9nq-master-1         Ready    master   20d     v1.17.1
ocp-jb9nq-master-2         Ready    master   20d     v1.17.1
ocp-jb9nq-worker-0-pxsfh   Ready    worker   17d     v1.17.1
ocp-jb9nq-worker-0-t48hm   Ready    worker   20d     v1.17.1
ocp-jb9nq-worker-0-w87sf   Ready    worker   20d     v1.17.1
```

Why would you want to have Infra nodes? Well, the simplest reason is to put your workloads on there that are not strictly part of the Control Plane, nor are they the Applications you want to run. The main items we mean when talking of such 'Infrastructure' is the handling of Routing, the Image Registry, Metrics, and Logging. Keeping these distinct from your applications gives a good separation of concerns, as well as the fact that Infra nodes don't incur subscription charges!

In order to create a new type of node, we need to create a new MachineSet. This can be done by expanding 'Compute', clicking on 'Machine Sets', then the 'Create Machine Set' button. You can copy the YAML for the existing worker MachineSet, and modify it by adding a label of `node-role.kubernetes.io/infra: ""`, as well as modifying the role and type of the node to be infra. My config is the below:

```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  labels:
    machine.openshift.io/cluster-api-cluster: ocp-jb9nq 
  name: ocp-jb9nq-infra-0
  namespace: openshift-machine-api
spec:
  replicas: 3
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: ocp-jb9nq 
      machine.openshift.io/cluster-api-machineset: ocp-jb9nq-infra-0
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: ocp-jb9nq 
        machine.openshift.io/cluster-api-machine-role: infra 
        machine.openshift.io/cluster-api-machine-type: infra 
        machine.openshift.io/cluster-api-machineset: ocp-jb9nq-infra-0 
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/infra: "" 
      providerSpec:
        value:
          cluster_id: 652f7152-98e1-11ea-9fa7-901b0e33b3aa
          userDataSecret:
            name: worker-user-data
          name: ''
          credentialsSecret:
            name: ovirt-credentials
          metadata:
            creationTimestamp: null
          template_name: ocp-jb9nq-rhcos
          kind: OvirtMachineProviderSpec
          id: ''
          apiVersion: ovirtproviderconfig.openshift.io/v1beta1
```

This will create 3 replicas of my new MachineSet. Saving this file and waiting a few minutes will see RHV spin up some new VMs, assign them as Nodes and complete the configuration. If we run `oc get nodes` again, we will see the following.

```bash
NAME                       STATUS   ROLES            AGE     VERSION
ocp-jb9nq-infra-0-2zrvd    Ready    infra, worker    3d23h   v1.17.1
ocp-jb9nq-infra-0-rrppr    Ready    infra, worker    3d23h   v1.17.1
ocp-jb9nq-infra-0-zq5cd    Ready    infra, worker    4d      v1.17.1
ocp-jb9nq-master-0         Ready    master           20d     v1.17.1
ocp-jb9nq-master-1         Ready    master           20d     v1.17.1
ocp-jb9nq-master-2         Ready    master           20d     v1.17.1
ocp-jb9nq-worker-0-pxsfh   Ready    worker           17d     v1.17.1
ocp-jb9nq-worker-0-t48hm   Ready    worker           20d     v1.17.1
ocp-jb9nq-worker-0-w87sf   Ready    worker           20d     v1.17.1
```

So now we have 3 Infra Nodes, and just need something to run on them. The first thing we can do is move the `IngressController`. We need to edit it and modify the `spec` section to add a `NodeSelector` stanza in the following format

```bash
oc edit ingresscontroller default -n openshift-ingress-operator -o yaml
```

replace `spec: {}` with the following - it will ensure that pods go on a node labeled as `infra`

```yaml
spec:
    nodePlacement:
      nodeSelector:
        matchLabels:
          node-role.kubernetes.io/infra: ""
```

You can confirm the correct nodes are in the right place by running

```bash
oc get pod -n openshift-ingress -o wide
```

To move the default registry, it is a similar pattern - first by editing the `config/instance` object

```bash
oc edit config/cluster
```

And adding the following in the `spec`

```yaml
nodeSelector:
    node-role.kubernetes.io/infra: ""
```

Once again, this can be verified by running

```bash
oc get pods -o wide -n openshift-image-registry
```

To move monitoring, we need to create a ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    grafana:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    openshiftStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
```

Applying the above can be done ising `oc create`

```bash
oc create -f cluster-monitoring-configmap.yaml
```

After a few minutes, you can check that the pods are in the correct place with the following command

```bash
oc get pod -n openshift-monitoring -o wide
```

The [OpenShift documentation](https://docs.openshift.com/container-platform/4.4/machine_management/creating-infrastructure-machinesets.html#infrastructure-moving-logging_creating-infrastructure-machinesets) for moving cluster logging resources is quite comprehensive, and worth following (as it is for the moving of other resources described above).

You may notice then when creating the nodes, they will all have the role of `infra, worker`. This does mean that there is the possibility that Application workloads could get put on the nodes. If you want to remove the worker label, `oc label node <node name> node-role.kubernetes.io/worker-` will do it for you.
