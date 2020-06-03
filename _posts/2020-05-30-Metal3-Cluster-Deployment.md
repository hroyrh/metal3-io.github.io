---
title: "Metal³ development environment walkthrough part 2: Deploying a new baremetal cluster"
draft: false
categories: ["metal3", "kubernetes", "cluster API", "metal3-dev-env"]
author: Himanshu Roy
---

## Introduction

This blog post describes how to deploy a bare metal cluster, a virtual one for simplicity, using [Metal³](https://github.com/metal3-io/metal3-dev-env). We will briefly discuss the steps involved in setting up the cluster as well as some of the customizations available. If you want to know more about the architechture of Metal³, this [blogpost](https://metal3.io/blog/2020/02/27/talk-kubernetes-finland-metal3.html) can be helpful.

This post builds upon the detailed metal3-dev-env walkthrough that describes in detail the steps involved in the environment set up and management cluster configuration. Here we will use that environment to deploy a new Kubernetes cluster using Metal³.
Before we get started, there are a couple of requirements we are expecting to be fulfilled.


## Requirements

- Metal³ is already deployed and working,  if not please follow the instructions in this [blog post](https://metal3.io/blog/2020/02/18/metal3-dev-env-install-deep-dive.html).
- The appropriate environment variables are setup via shell or in the config_${user}.sh file, for example -
  - CAPI_VERSION
  - NUM_NODES
  - CLUSTER_NAME
- The host you are planning to use here has sufficient resources ( for a single node cluster )
  - At least 4096M memory
  - At least 2 vCpu

## BareMetal Cluster Deployment

The deployment scripts primarily use ansible and the existing Kubernetes management cluster (based on minikube ) for deploying the bare-metal cluster. There is an option to specify some features of the new cluster via environment-variables, such as :

| Parameter           | Description                  | Default                  |
| ------------------- | ---------------------------- | ------------------------ |
| CAPI_VERSION        | Version of Metal3 API4       | v1alpha3/v1alpha4        |
| POD_CIDR            | Pod Network CIDR             | 192.168.0.0/18           |
| CLUSTER_NAME        | Name of bare metal cluster   | test1                    |


### Steps

All the scripts for cluster provisioning or deprovisioning are located at - [`${metal3-dev-env}/scripts/v1alphaX/`](https://github.com/metal3-io/metal3-dev-env/tree/master/scripts/v1alphaX). The scripts call a common playbook which handles all the tasks that are available.
The steps involved in the process are :

- Some of the configuration files generated as part of provisioning a cluster, in a centos based environment, are :

| Name           | Description                                       | Path                          |
| ------------------- | ---------------------------------------------| ----------------------------- |
| clusterctl env file        | Contains Cluster Environment details  | `${Manifests}/clusterctl_env_centos.rc` |
| POD_CIDR            | Pod Network CIDR                             |  |
| CLUSTER_NAME        |  Name of bare metal cluster                  |  |

  - clusterctl env file : `${Manifests}/clusterctl_env_centos.rc`
  - manifest file for cluster is generated using clusterctl-tool : `${Manifests}/manifests.yaml`
  - directory for kustomization resource files : `${Manifests}/base`
    > Note : the `base` directory has templates for the resources that are created while provisioning
  - [kustomization](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) file : `${Maifests}/kustomization.yaml`
  - [`kustomize-tool`](https://github.com/kubernetes-sigs/kustomize) is used to build manifests in the base directory
  - cluster, controlplane and machinedeployment manifests are generated in `base` directory
  - worker manifests are generated and added using kustomize-tool
> **Note** :  *All the resources created as part of provisioning have their templates under the ‘base’ directory*
- The script calls an ansible playbook with necessary parameter ( from env vars and defaults )
- The playbook executes the role -, [`${metal3-dev-env}/vm-setup/roles/v1aX_integration_test`](https://github.com/metal3-io/metal3-dev-env/tree/master/vm-setup/roles/v1aX_integration_test), which runs the corresponding [task_file](https://github.com/metal3-io/metal3-dev-env/tree/master/vm-setup/roles/v1aX_integration_test/tasks) for provisioning/deprovisioning the cluster/controlplane or a worker
- There are [templates](https://github.com/metal3-io/metal3-dev-env/tree/master/vm-setup/roles/v1aX_integration_test/templates) in the role, which are copied to the `manifest` directory. They are used for generating configuration for the cluster and kubeadm, which is then supplied to the Kubernetes module of ansible to create the cluster.
- During provisioning, first the `clusterctl` env file is generated, then and `manifest` file is created/updated using the `clusterctl` tool, which is then supplied to `kustomize` tool to create the config files for provisioning the resource, from the available templates.
- The manifests are generated for - cluster, controlplane and machinedeployment, and the worker manifests are merged with the deployment manifests.
- Finally a yaml file is generated with all the details for the cluster/controlplane/worker, which is then passed to K8s module in ansible which creates the resources.
- There are also manifests stored in ${Manifests} directory - [`${metal3-dev-env}/vm-setup/roles/v1aX_integration_test/files/manifests`](https://github.com/metal3-io/metal3-dev-env/tree/master/vm-setup/roles/v1aX_integration_test), for the cluster which can be used to deprovision the cluster at a later point in time.
> **Note** : * the `manifests` directory is not present but created when you first use the scripts.*
- Centos or Ubuntu images [are downloaded](https://github.com/metal3-io/metal3-dev-env/blob/master/vm-setup/roles/v1aX_integration_test/tasks/download_image.yml) when provisioning a controlplane or a worker

**Note** : *Here is a depiction of the common steps, mainly involving generating templates and config files.*

![A diagram depicting the Generate Templates workflow](/assets/images/metal3-generate-templates.svg)


### Provision Cluster
This script, located at the path - `${metal3-dev-env}/scripts/v1alphaX/provision_clusters.sh`, provisions the cluster. This script creates a `Metal3Cluster` resource. To see if you have a successful deployment, just do - 
```console
kubectl get metal3cluster ${CLUSTER_NAME} -n metal3
```
This will return the cluster deployed, and you can check the cluster details by describing the returned resource.

Here is what a `Cluster` resource looks like :
```console
kubectl describe Cluster ${CLUSTER_NAME} -n metal3
```
```yaml
Name:         eko
Namespace:    metal3
Labels:       <none>
Annotations:  <none>
API Version:  cluster.x-k8s.io/v1alpha3
Kind:         Cluster
Metadata:
  Creation Timestamp:  2020-05-22T18:19:54Z
  Finalizers:
    cluster.cluster.x-k8s.io
  Generation:  2
  Managed Fields:
    API Version:  cluster.x-k8s.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:clusterNetwork:
          .:
          f:pods:
            .:
            f:cidrBlocks:
          f:services:
            .:
            f:cidrBlocks:
        f:controlPlaneRef:
          .:
          f:apiVersion:
          f:kind:
          f:name:
        f:infrastructureRef:
          .:
          f:apiVersion:
          f:kind:
          f:name:
    Manager:      OpenAPI-Generator
    Operation:    Update
    Time:         2020-05-22T18:19:54Z
    API Version:  cluster.x-k8s.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
          .:
          v:"cluster.cluster.x-k8s.io":
      f:spec:
        f:controlPlaneEndpoint:
          f:host:
          f:port:
      f:status:
        .:
        f:infrastructureReady:
        f:phase:
    Manager:         manager
    Operation:       Update
    Time:            2020-05-22T18:19:54Z
  Resource Version:  5795164
  Self Link:         /apis/cluster.x-k8s.io/v1alpha3/namespaces/metal3/clusters/eko
  UID:               b1e342ee-7f53-4dca-8959-8665aa74a26a
Spec:
  Cluster Network:
    Pods:
      Cidr Blocks:
        192.168.0.0/18
    Services:
      Cidr Blocks:
        10.96.0.0/12
  Control Plane Endpoint:
    Host:  192.168.111.249
    Port:  6443
  Control Plane Ref:
    API Version:  controlplane.cluster.x-k8s.io/v1alpha3
    Kind:         KubeadmControlPlane
    Name:         eko
    Namespace:    metal3
  Infrastructure Ref:
    API Version:  infrastructure.cluster.x-k8s.io/v1alpha4
    Kind:         Metal3Cluster
    Name:         eko
    Namespace:    metal3
Status:
  Infrastructure Ready:  true
  Phase:                 Provisioned
Events:                  <none>
```

### Provision Controlplane

This script, located at the path - ${metal3-dev-env}/scripts/v1alphaX/provision_clusters.sh, provisions the controlplane or the master of the cluster. As part of the controlplane provisioning, the generated manifests are for cluster, controlplane and machinedeployment ( creates a MachineSet, which creates a Machine that manages the master node ). It creates a `KubeadmControlPlane` resource and a `Metal3MachineTemplate` resource. To check if you have a successful deployment, try this - 
```console
kubectl get KubeadmControlPlane ${CLUSTER_NAME} -n metal3
kubectl describe KubeadmControlPlane ${CLUSTER_NAME} -n metal3
```

```yaml
Name:         eko
Namespace:    metal3
Labels:       cluster.x-k8s.io/cluster-name=eko
Annotations:  <none>
API Version:  controlplane.cluster.x-k8s.io/v1alpha3
Kind:         KubeadmControlPlane
Metadata:
  Creation Timestamp:  2020-05-22T19:05:48Z
  Finalizers:
    kubeadm.controlplane.cluster.x-k8s.io
  Generation:  1
  Managed Fields:
    API Version:  controlplane.cluster.x-k8s.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:infrastructureTemplate:
          .:
          f:apiVersion:
          f:kind:
          f:name:
        f:kubeadmConfigSpec:
          .:
          f:files:
          f:initConfiguration:
            .:
            f:nodeRegistration:
              .:
              f:kubeletExtraArgs:
                .:
                f:node-labels:
              f:name:
          f:joinConfiguration:
            .:
            f:controlPlane:
            f:nodeRegistration:
              .:
              f:kubeletExtraArgs:
                .:
                f:node-labels:
              f:name:
          f:postKubeadmCommands:
          f:preKubeadmCommands:
          f:users:
        f:replicas:
        f:version:
    Manager:      OpenAPI-Generator
    Operation:    Update
    Time:         2020-05-22T19:05:48Z
    API Version:  controlplane.cluster.x-k8s.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
          .:
          v:"kubeadm.controlplane.cluster.x-k8s.io":
        f:labels:
          .:
          f:cluster.x-k8s.io/cluster-name:
        f:ownerReferences:
          .:
          k:{"uid":"b1e342ee-7f53-4dca-8959-8665aa74a26a"}:
            .:
            f:apiVersion:
            f:blockOwnerDeletion:
            f:controller:
            f:kind:
            f:name:
            f:uid:
      f:status:
        .:
        f:replicas:
        f:selector:
        f:unavailableReplicas:
        f:updatedReplicas:
    Manager:    manager
    Operation:  Update
    Time:       2020-05-22T19:06:05Z
  Owner References:
    API Version:           cluster.x-k8s.io/v1alpha3
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Cluster
    Name:                  eko
    UID:                   b1e342ee-7f53-4dca-8959-8665aa74a26a
  Resource Version:        5811363
  Self Link:               /apis/controlplane.cluster.x-k8s.io/v1alpha3/namespaces/metal3/kubeadmcontrolplanes/eko
  UID:                     e599039c-a72f-40a2-8c63-9126a1627527
Spec:
  Infrastructure Template:
    API Version:  infrastructure.cluster.x-k8s.io/v1alpha4
    Kind:         Metal3MachineTemplate
    Name:         eko-controlplane
    Namespace:    metal3
  Kubeadm Config Spec:
    Files:
      Content:  ! Configuration File for keepalived
global_defs {
    notification_email {
    sysadmin@example.com
    support@example.com
    }
    notification_email_from lb@example.com
    smtp_server localhost
    smtp_connect_timeout 30
}
vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 1
    priority 101
    advert_int 1
    virtual_ipaddress {
        192.168.111.249
    }
}

      Path:     /etc/keepalived/keepalived.conf
      Content:  BOOTPROTO=dhcp
DEVICE=eth1
ONBOOT=yes
TYPE=Ethernet
USERCTL=no

      Owner:        root:root
      Path:         /etc/sysconfig/network-scripts/ifcfg-eth1
      Permissions:  0644
      Content:      [kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

      Owner:        root:root
      Path:         /etc/yum.repos.d/kubernetes.repo
      Permissions:  0644
    Init Configuration:
      Local API Endpoint:
        Advertise Address:  
        Bind Port:          0
      Node Registration:
        Kubelet Extra Args:
          Node - Labels:  metal3.io/uuid={{ ds.meta_data.uuid }}
        Name:             {{ ds.meta_data.name }}
    Join Configuration:
      Control Plane:
        Local API Endpoint:
          Advertise Address:  
          Bind Port:          0
      Discovery:
      Node Registration:
        Kubelet Extra Args:
          Node - Labels:  metal3.io/uuid={{ ds.meta_data.uuid }}
        Name:             {{ ds.meta_data.name }}
    Post Kubeadm Commands:
      mkdir -p /home/metal3/.kube
      cp /etc/kubernetes/admin.conf /home/metal3/.kube/config
      chown metal3:metal3 /home/metal3/.kube/config
    Pre Kubeadm Commands:
      ifup eth1
      dnf update -y
      dnf install -y ebtables socat conntrack-tools
      dnf install python3 -y
      dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
      setenforce 0
      sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
      dnf install docker-ce docker-ce-cli --disableexcludes=kubernetes --nobest -y
      dnf install gcc kernel-headers kernel-devel keepalived device-mapper-persistent-data lvm2 -y
      echo  "Installing kubernetes binaries"
      curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/{kubeadm,kubelet,kubectl}
      chmod a+x kubeadm kubelet kubectl
      mv kubeadm kubelet kubectl /usr/local/bin/
      mkdir -p /etc/systemd/system/kubelet.service.d
      curl -sSL "https://raw.githubusercontent.com/kubernetes/release/v0.2.7/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:/usr/local/bin:g" > /etc/systemd/system/kubelet.service
      curl -sSL "https://raw.githubusercontent.com/kubernetes/release/v0.2.7/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:/usr/local/bin:g" > /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
      usermod -aG docker metal3
      systemctl enable --now docker keepalived kubelet
    Users:
      Name:  metal3
      Ssh Authorized Keys: ${SSH_KEY}
      Sudo:  ALL=(ALL) NOPASSWD:ALL
  Replicas:  1
  Version:   v1.18.0
Status:
  Replicas:              1
  Selector:              cluster.x-k8s.io/cluster-name=eko,cluster.x-k8s.io/control-plane=
  Unavailable Replicas:  1
  Updated Replicas:      1
Events:                  <none>
```

```console
kubectl get Metal3MachineTemplate ${CLUSTER_NAME}-controlplane -n metal3 
kubectl get bmh -n metal3 -w   > You should see all the nodes that were created at the time of metal3 deployment, along with their current status as the provisioning progresses
```

> Once the provisioning is finished, let's get the host-ip : 
```console
sudo virsh net-dhcp-leases baremetal
```
> **Note** : *baremetal is one of the 2 networks that were created at the time of Metal3 deployment, the other being “provisioning” which is used - as you have guessed - for provisioning the bare metal cluster. More details about networking setup in the metal3-dev-env environment are described in a previous [blog post](http://baremetal).*

You can login to the master node if you want, and can check the deployment status
```console
ssh metal3@{master-node-ip}
```

### Provision Workers

The script is located at ${metal3-dev-env-path}/scripts/v1alphaX/provision_worker.sh and it provisions a node to be added as a worker to the baremetal cluster. It selects one of the remaining nodes and provisions it and adds it to the baremetal cluster ( which only has a master at this point ). The resources created for workers are - `MachineDeployment` which can be scaled up to add more workers to the cluster, and it also created a `MachineSet` which then creates a `Machine` managing the node.

This is what a `MachineDeployment` looks like
```console
kubectl describe MachineDeployment ${CLUSTER_NAME} -n metal3
```
```yaml
Name:         eko
Namespace:    metal3
Labels:       cluster.x-k8s.io/cluster-name=eko
              nodepool=nodepool-0
Annotations:  machinedeployment.clusters.x-k8s.io/revision: 1
API Version:  cluster.x-k8s.io/v1alpha3
Kind:         MachineDeployment
Metadata:
  Creation Timestamp:  2020-05-27T15:54:28Z
  Generation:          1
  Managed Fields:
    API Version:  cluster.x-k8s.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .:
          f:cluster.x-k8s.io/cluster-name:
          f:nodepool:
      f:spec:
        .:
        f:clusterName:
        f:replicas:
        f:selector:
          .:
          f:matchLabels:
            .:
            f:cluster.x-k8s.io/cluster-name:
            f:nodepool:
        f:template:
          .:
          f:metadata:
            .:
            f:labels:
              .:
              f:cluster.x-k8s.io/cluster-name:
              f:nodepool:
          f:spec:
            .:
            f:bootstrap:
              .:
              f:configRef:
                .:
                f:apiVersion:
                f:kind:
                f:name:
            f:clusterName:
            f:infrastructureRef:
              .:
              f:apiVersion:
              f:kind:
              f:name:
            f:version:
    Manager:      OpenAPI-Generator
    Operation:    Update
    Time:         2020-05-27T15:54:28Z
    API Version:  cluster.x-k8s.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:machinedeployment.clusters.x-k8s.io/revision:
        f:ownerReferences:
      f:status:
        .:
        f:observedGeneration:
        f:phase:
        f:replicas:
        f:selector:
        f:unavailableReplicas:
        f:updatedReplicas:
    Manager:    manager
    Operation:  Update
    Time:       2020-05-27T15:54:33Z
  Owner References:
    API Version:     cluster.x-k8s.io/v1alpha3
    Kind:            Cluster
    Name:            eko
    UID:             ae0f4084-b33e-423c-96f2-ad72e3038e67
  Resource Version:  76157
  Self Link:         /apis/cluster.x-k8s.io/v1alpha3/namespaces/metal3/machinedeployments/eko
  UID:               a6597451-356c-46bb-be41-f93e9a4b6e4d
Spec:
  Cluster Name:               eko
  Min Ready Seconds:          0
  Progress Deadline Seconds:  600
  Replicas:                   1
  Revision History Limit:     1
  Selector:
    Match Labels:
      cluster.x-k8s.io/cluster-name:  eko
      Nodepool:                       nodepool-0
  Strategy:
    Rolling Update:
      Max Surge:        1
      Max Unavailable:  0
    Type:               RollingUpdate
  Template:
    Metadata:
      Labels:
        cluster.x-k8s.io/cluster-name:  eko
        Nodepool:                       nodepool-0
    Spec:
      Bootstrap:
        Config Ref:
          API Version:  bootstrap.cluster.x-k8s.io/v1alpha3
          Kind:         KubeadmConfigTemplate
          Name:         eko-workers
      Cluster Name:     eko
      Infrastructure Ref:
        API Version:  infrastructure.cluster.x-k8s.io/v1alpha4
        Kind:         Metal3MachineTemplate
        Name:         eko-workers
      Version:        v1.18.0
Status:
  Observed Generation:   1
  Phase:                 ScalingUp
  Replicas:              1
  Selector:              cluster.x-k8s.io/cluster-name=eko,nodepool=nodepool-0
  Unavailable Replicas:  1
  Updated Replicas:      1
Events:                  <none>
```

To check the status we can follow some of the previous steps :
`kubectl get bmh -n metal3 -w`   > To see the live status of the node being provisioned

We can also see the tag of something like - ${cluster-name}-worker-$id
`sudo virsh net-dhcp-leases baremetal`   > To get the nodes IP

`ssh metal3@{master-node-ip}`   > To check if its added to the cluster

`ssh metal3@{node-ip}`   > If you want to login to the node

We can add or remove workers to the cluster, we can scale up the MachineDeployment up or down, in this example we are adding 2 more worker nodes
`kubectl scale --replicas=3 MachineDeployment ${CLUSTER_NAME} -n metal3`


### Deprovisioning

All of the previous components have corresponding deprovisioning scripts which use config files, in the previously mentioned manifest directory, and use them to clean up worker, controlplane and cluster.


