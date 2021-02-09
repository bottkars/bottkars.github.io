---
layout: post
title: "Protecting AKS Workloads on AzureStackHub Part 1"
description: "create, snapshot and clone persistent volumes the k8s way"
modified: 2021-02-09
comments: true
published: true
tags: [aks-engine, json, bash, csi, aks, powerprotect]
image:
  path: /images/masterpipeline_ppdm_aks.png
  feature: masterpipeline_ppdm_aks.png
  credit: 
  creditlink: 
---
# Protecting AKS Workloads on AzureStackHub Part 1

If you have read my previous article, you could get a Brief understanding how we can Protect AKS Persistent Workloads on Azure using @ProjectVelero and DellEMC PowerProtect Datamanager.

One of my personal aspirations is always: 
*If it runs on Azure, it should run on AzureStackHub*

Well, we all know, AzureStackHub is like Azure, *but different*

## What works, how it works and what is/was missing

Before we start, let´s get some basics.

### What *was* missing

AzureStackHub allows you to deploy Kubernetes Clusters using the AKS Engine.
AKS Engine is a legacy tool to create ARM Template´s to deploy Kubernetes Clusters.
While Public Azure AKS Clusters will transition to Cluster API ( CAPZ ), AzureStack Hub only support AKS-Engine
The Current ( officially Supported ) version of AKS-Engine for AzureStack Hub is v0.55.4.

It allows for Persistent Volumes, however, they would use the InTree Volume Plugin.
In order to make use of the Container Storage Interface (CSI), we first would need a CSI Driver that is able to talk to AzureStackHub.
When I tried to implement the Azure CSI Drivers on AzureStack Hub last year, I essentially failed because of a ton of Certificate and API Issues.

With PowerProtect official Support for Azure, I started to dig into the CSI Drivers again.
I browsed through the existing Github Issues and PR´s, and found at least that some People are working on iot.

And finally a got in touch with Andy Zhang. who maintains the azuredisk-csi-driver kuberenetes-sigs.
From an initial "it should" work, he connected me to the people doing E2E Test for AzureStackHub.

Within 2 Days turnaround, we managed to fix all API and SSL related issues, and *FINALLY GOT A WORKING VERSION* !

### how it works

I am not going to explain how to deploy AKS-Engine based Clusters on AzureStackHub, there is a good explanation on the [Microsoft Documentation](https://docs.microsoft.com/en-us/azure-stack/user/azure-stack-kubernetes-aks-engine-overview?view=azs-2008#:~:text=The%20AKS%20engine%20provides%20a%20command-line%20tool%20to,other%20infrastructure-as-a-service%20(IaaS)%20resources%20in%20Azure%20Stack%20Hub.) Website.


Once you Cluster is deployed, you need to deploy *the latest* azuredisk-csi-drivers.

Microsoft Provides a guidance here that *helm charts* must be used to deploy the azuredisk-csi-drivers on AzureStackHub.
#### installing the driver
So first we add the Repo from Github:

{% highlight shell %}
helm repo add azuredisk-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/charts
{% endhighlight %}

With the Repo now added, we can now deploy the azuredisk-csi-driver Helm Chart.
When doing this, we will pass some setting to the deployment:
 - cloud=AzureStackCloud
This determines we run on AzureSTackHub and instructs the csi driver to load the Cloud Config from a File on the Master.

- snapshot.enabled=true
This installs the *csi-snapshot-controller* that is required to expose Snapshot Functionality

We deploy the driver with:

{% highlight shell %}
helm install azuredisk-csi-driver azuredisk-csi-driver/azuredisk-csi-driver \
--namespace kube-system \
--set cloud=AzureStackCloud \
--set snapshot.enabled=true
{% endhighlight %}


<figure class="full">
	<img src="/images/helm_install.png" alt="">
	<figcaption>installing</figcaption>
</figure>
This should install:
A Replica Set for the csi-azuredisk-controller with 2 Pods containing the following containers:
	mcr.microsoft.com/k8s/csi/azuredisk-csi
	mcr.microsoft.com/oss/kubernetes-csi/csi-attacher
	mcr.microsoft.com/oss/kubernetes-csi/csi-provisioner
	mcr.microsoft.com/oss/kubernetes-csi/csi-resizer
	mcr.microsoft.com/oss/kubernetes-csi/csi-snapshotter
	mcr.microsoft.com/oss/kubernetes-csi/livenessprobe

A Replica Set for the csi-snapshot-controller with 1 Pod
One csi-azuredisk-node Pod per Node
and the corresponding CRD´s for the snapshotter

you can check the pods with
{% highlight shell %}
kubectl -n kube-system get pod -o wide --watch -l app=csi-azuredisk-controller
kubectl -n kube-system get pod -o wide --watch -l app=csi-azuredisk-node
{% endhighlight %}

<figure class="full">
	<img src="/images/csi_helm.png" alt="">
	<figcaption>CSI Helm CHart</figcaption>
</figure>


#### Adding the Storageclasses

When AKS is deployed using the Engine, most likely 3 Storageclasses are installed by the In-Tree Provider

<figure class="full">
	<img src="/images/aks_storageclasses.png" alt="">
	<figcaption>AKS Storageclasses</figcaption>
</figure>


In order to make use of the CSI Storageclass, we need to add at least one new Storageclass:
create a class_csi.yaml with the following content

{% highlight yaml %}
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi
provisioner: disk.csi.azure.com
parameters:
  skuname: Standard_LRS  # alias: storageaccounttype, available values: Standard_LRS, Premium_LRS
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
{% endhighlight %}


and then run 

{% highlight shell %}
kubectl apply -f class_csi.yaml
{% endhighlight %}

and check with 

{% highlight shell %}
kubectl get storageclasses
{% endhighlight %}

#### optional: Add a snapshot Class

Similar to the Storage Class, we may want to add a Snaphot Class
apply the below config with

{% highlight shell %}
kubectl apply -f storageclass-azuredisk-snapshot.yaml
{% endhighlight %}


{% highlight yaml %}
---
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: csi-azuredisk-vsc
driver: disk.csi.azure.com
deletionPolicy: Delete
parameters:
  incremental: "false"  # available values: "true", "false" ("true" by default for Azure Public Cloud, and "false" by de
fault for Azure Stack Cloud)
{% endhighlight %}



### Testing stuff works

Following the Microsoft Documentation, create a Statefulset with Azure Disk Mount:
{% highlight shell %}
kubectl create -f https://raw.githubusercontent.com/kubernetes-sigs/azuredisk-csi-driver/master/deploy/example/statefulset.yaml
{% endhighlight %}

Verify the deployment with 

{% highlight shell %}
kubectl -n default describe StatefulSet/statefulset-azuredisk
kubectl get pvc
{% endhighlight %}


<figure class="full">
	<img src="/images/pvc_cli.png" alt="">
	<figcaption>PVC from CLI</figcaption>
</figure>


The PVC will show the identical Volume Name as the Disk Name from The Portal / CLI

<figure class="full">
	<img src="/images/pvc_cli.png" alt="">
	<figcaption>AKS Storageclasses</figcaption>
</figure>


You know should have a Running azuredisk-csi-driver Environment on your AzureSTack Hub. 
Stay Tuned for Part 2 including DataProtection with PowerProtect Datamanager ...