---
title: IBM Cloud
logo: /logos/ibm.png
linkTitle: IBM Cloud
weight: 5
keywords: portworx, IBM, kubernetes, PaaS, IaaS, docker, converged, cloud, IKS
description: Deploy Portworx on IBM Cloud Kubernetes Service. See for yourself how easy it is!
noicon: true
---

This guide shows how you can deploy Portworx on an [IBM Cloud Kubernetes Service](https://www.ibm.com/cloud/container-service) or [Red Hat OpenShift on IBM Cloud](https://www.ibm.com/cloud/redhat) cluster.

## Prerequisites

Before you begin:

- [Sign up for an IBM Cloud Pay-As-You-Go account](https://cloud.ibm.com/registration). With an IBM Cloud Pay-As-You-Go account you can access the IBM Cloud Platform-as-a-Service and Infrastructure-as-a-Service portfolio.
- Learn about [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-cs_ov#cs_ov) or [Red Hat OpenShift on IBM Cloud](https://cloud.ibm.com/docs/openshift?topic=openshift-why_openshift) and the service benefits.

## Step 1: Plan the setup of your IBM Cloud Kubernetes Service cluster

Portworx is a highly available software-defined storage solution that you can use to manage persistent storage for your containerized databases or other stateful apps in your IBM Cloud Kubernetes Service cluster across multiple zones. To make sure that your cluster is set up with the compute resources that are required for Portworx, review the FAQs in this step.

**What worker node flavor in IBM Cloud Kubernetes Service is the right one for Portworx?** </br>
IBM Cloud Kubernetes Service provides [bare metal worker node flavors that are optimized for software-defined storage (SDS) usage](https://cloud.ibm.com/docs/containers?topic=containers-plan_clusters#sds) and that come with one or more raw, unformatted, and unmounted local disks that you can use for your Portworx storage layer. Portworx offers best performance when you use SDS worker node machines that come with 10Gbps network speed.

**What if I want to run Portworx on non-SDS worker nodes?** </br>
You can install Portworx on non-SDS worker node flavors, but you might not get the performance benefits that your app requires. Non-SDS worker nodes can be virtual or bare metal. If you want to use [virtual machines](https://cloud.ibm.com/docs/containers?topic=containers-plan_clusters#vm), use a worker node flavor of `b3c.16x64` or better. Virtual machines with a flavor of `b3c.4x16` or `u3c.2x4` do not provide the required resources for Portworx to work properly. [Bare metal machines](https://cloud.ibm.com/docs/containers?topic=containers-plan_clusters#bm) come with sufficient compute resources and network speed for Portworx. For more information about the compute resources that are required by Portworx, see the [minimum requirements](/start-here-installation/#installation-prerequisites).

To add non-SDS worker nodes to the Portworx storage layer, each worker node must have at least one tertiary raw, unformatted, and unmounted disk that is attached to the worker node. You can manually add these tertiary disks or use the [IBM Cloud Block Attacher plug-in](https://cloud.ibm.com/docs/containers?topic=containers-utilities#block_storage_attacher) to automatically add the disks to your non-SDS worker nodes. For more information, see the [IBM Cloud Kubernetes Service documentation](https://cloud.ibm.com/docs/containers?topic=containers-portworx#create_block_storage).

**How can I make sure that my data is stored highly available?** </br>
You need at least 3 worker nodes in your Portworx cluster so that Portworx can replicate your data across nodes. By replicating your data across worker nodes, Portworx can ensure that your stateful app can be rescheduled to a different worker node in case of a failure without losing data. For even higher availability, use a [multizone cluster](https://cloud.ibm.com/docs/containers?topic=containers-plan_clusters#multizone) and replicate your volumes on SDS worker nodes across 3 zones.

**Can I install Portworx on all supported infrastructure providers and container platforms?** </br>
Portworx is supported only in classic IBM Cloud Kubernetes Service and Red Hat OpenShift on IBM Cloud clusters, and is not available in VPC on Classic clusters.

## Step 2: Create an IBM Cloud Kubernetes Service cluster

To install Portworx, you must have an IBM Cloud Kubernetes Service or Red Hat OpenShift on IBM Cloud cluster. 

1. Plan the network setup for your [Kubernetes](https://cloud.ibm.com/docs/containers?topic=containers-plan_clusters) or [OpenShift](https://cloud.ibm.com/docs/openshift?topic=openshift-plan_clusters) cluster. 
2. Decide if you want to create a multizone [Kubernetes](https://cloud.ibm.com/docs/containers?topic=containers-plan_clusters#multizone) or [OpenShift](https://cloud.ibm.com/docs/openshift?topic=openshift-ha_clusters#multizone) cluster for high availability. If you do, you must enable [VLAN spanning](https://cloud.ibm.com/docs/infrastructure/vlans?topic=vlans-vlan-spanning#vlan-spanning) or [Virtual Routing and Forwarding (VRF)](https://cloud.ibm.com/docs/infrastructure/direct-link?topic=direct-link-overview-of-virtual-routing-and-forwarding-vrf-on-ibm-cloud#overview-of-virtual-routing-and-forwarding-vrf-on-ibm-cloud) for your IBM Cloud account.  
3. Choose the worker node flavor for your [Kubernetes](https://cloud.ibm.com/docs/containers?topic=containers-planning_worker_nodes) or [OpenShift](https://cloud.ibm.com/docs/openshift?topic=openshift-planning_worker_nodes) cluster that you want to use. Make sure to review Step 1 to find frequently asked questions for the type of worker node that best meets the Portworx minimum requirements. 
4. Make sure that your IBM Cloud account is set up with the right [permissions](https://cloud.ibm.com/docs/containers?topic=containers-clusters#cluster_prepare) to create a cluster. 
5. Create a [Kubernetes](https://cloud.ibm.com/docs/containers?topic=containers-clusters#clusters_ui) or [OpenShift](https://cloud.ibm.com/docs/openshift?topic=openshift-openshift-create-cluster#openshift_create_cluster_console) cluster. 
6. If you created a cluster with non-SDS worker nodes, [add raw, unformatted, and unmounted block storage](https://cloud.ibm.com/docs/containers?topic=containers-portworx#create_block_storage) to your worker nodes. The block storage devices are attached to your worker node and can be included in the Portworx storage layer. If you used SDS worker nodes, you do not need to attach raw block storage devices to your worker nodes. 

## Step 3: Set up a key-value store for the Portworx metadata

Every Portworx cluster must be connected to a key-value store to store Portworx metadata. The Portworx key-value store serves as the single source of truth for your Portworx storage layer. If the key-value store is not available, then you cannot work with your Portworx cluster to access or store your data. Existing data is not changed or removed when the Portworx database is unavailable.

In order for your Portworx cluster to be highly available, you must ensure that the Portworx key-value store is set up to be highly available. By using an instance of [Databases for etcd](https://cloud.ibm.com/docs/containers?topic=containers-portworx#portworx_database), you can set up a highly available key-value store for your Portworx cluster. Each service instance contains three etcd data members that are added to a cluster. The etcd data members are spread across zones in an IBM Cloud location and data is replicated across all etcd data members.

## Step 4: Set up volume encryption with IBM Key Protect

By default, data that you store on a Portworx volume is not encrypted at rest or during transit. To protect your data from being accessed by unauthorized users, you can choose to protect your volumes with [IBM Key Protect](https://cloud.ibm.com/docs/services/key-protect?topic=key-protect-about#about). IBM Key Protect helps you to provision encrypted keys that are secured by FIPS 140-2 Level 2 certified cloud-based hardware security modules (HSMs).

Review the following information in the IBM Cloud Kubernetes Service documentation:

- [IBM Key Protect volume encryption flow](https://cloud.ibm.com/docs/containers?topic=containers-portworx#encryption)
- [IBM Key Protect volume decryption flow](https://cloud.ibm.com/docs/containers?topic=containers-portworx#decryption)
- [Setting up IBM Key Protect encryption for your volumes](https://cloud.ibm.com/docs/containers?topic=containers-portworx#setup_encryption)

## Step 5: Install Portworx in your cluster

Install Portworx with a Helm chart. The Helm chart deploys a trial version of the Portworx enterprise edition `px-enterprise` that you can use for 30 days. After the trial version expires, you must [purchase a Portworx license](/reference/knowledge-base/px-licensing/) to continue to use your Portworx cluster. In addition, Stork is installed on your Kubernetes cluster. Stork is the Portworx storage scheduler and allows you to co-locate pods with their data, and create and restore snapshots of Portworx volumes.

Before you begin:

- Make sure that you set up your Portworx database to store your Portworx cluster metadata (Step 3).
- Decide if you want to enable Portworx volume encryption (Step 4).
- If you use non-SDS worker nodes, [add raw, unformatted, and unmounted block storage](https://cloud.ibm.com/docs/containers?topic=containers-portworx#create_block_storage) to your worker nodes.

For more information about how to install the Portworx Helm chart, see the [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-portworx#install_portworx) or [Red Hat OpenShift on IBM Cloud ](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#install_portworx) documentation.

## Step 6: Add Portworx storage to your apps

Now that your Portworx cluster is all set, you can start creating Portworx volumes by using [Kubernetes dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/). The Portworx Helm chart already set up a few default storage classes in your cluster that you can see by running the `kubectl get storageclasses | grep portworx` command. You can also create your own storage class to define settings, such as:

- Encryption for a volume
- IO priority of the disk where you want to store the data
- Number of data copies that you want to store across worker nodes
- Sharing of volumes across pods

For more information about how to create your own storage class and add Portworx storage to your app, see the [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-portworx#add_portworx_storage) or [Red Hat OpenShift on IBM Cloud](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#add_portworx_storage) documentation. For an overview of supported configurations in a PVC, see [Dynamic Provisioning of PVCs](/portworx-install-with-kubernetes/storage-operations/create-pvcs/dynamic-provisioning/).

## What's next?
Now that you set up Portworx on your IBM Cloud Kubernetes Service cluster, you can explore the following features:

- **Use existing Portworx volumes:** If you have an existing Portworx volume that you created manually or that was not automatically deleted when you deleted the PVC, you can statically provision the corresponding PV and PVC and use this volume with your app. For more information, see [Using pre-provisioned volumes](/portworx-install-with-kubernetes/storage-operations/create-pvcs/using-preprovisioned-volumes/).
- **Running stateful sets on Portworx:** If you have a stateful app that you want to deploy as a stateful set into your cluster, you can set up your stateful set to use storage from your Portworx cluster. For more information, see [Stateful applications](/portworx-install-with-kubernetes/storage-operations/kubernetes-storage-101/applications/).
- **Running your pods hyperconverged:** You can configure your Portworx cluster to schedule pods on the same worker node where the pod's volume resides. This setup is also referred to as hyperconverged and can improve the data storage performance. For more information, see [Run hyperconverged](/portworx-install-with-kubernetes/storage-operations/hyperconvergence/).
- **Creating snapshots of your Portworx volumes:** You can save the current state of a volume and its data by creating a Portworx snapshot. Snapshots can be stored on your local Portworx cluster or in the Cloud. For more information, see [Create snapshots](/portworx-install-with-kubernetes/storage-operations/create-snapshots/).
