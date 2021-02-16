---
layout: post
title: "up to date AKS Image SKU`s on AzureStack Hub, newer Versions of AKS-Engine and Kubernetes"
description: "stay up to date"
modified: 2021-02-16
comments: true
published: true
tags: [aks-engine, json, bash, csi, aks, AzureStackHub]
image:
  path: /images/aks_compute_images.png
  feature: aks_compute_images.png
  credit: 
  creditlink: 
---
# keep my K8S up to date with Azure... 

*Disclaimer: use this at your own Risk*

Recently I tried to deploy newer versions of Kubernetes using AKS Engine to my AzureStack hub, but i failed for some odd reasons. That made me thinking about how do get newer Versions deployed to my AzureStack Hub using aks-engine.   

Why should you want a newer version at all? Well, the answer is fairly simple. With K8s dot releases essentially coming monthly, one want's to stay current with Security fixes. Patching is not an option. I love the image based approach where i simply replace the OS Image. The longer a VM runs, the more vulnerable it get's. Deleting a VM an rebuild with a new Image is by far the most convenient and secure approach. aks-egine does a pretty good job in upgrading to newer image versions.... if available.
## The Problem

If you want to use your own Image, or try a newer AKS Engine release, the Deployment will try to download the missing Packages from https://kubernetesartifacts.azureedge.net/kubernetes/ and the according Docker Images from mcr.microsoft.com/oss/kubernetes/  


This Process is most likely to fail for slow Networks, as the download scripts have timeouts of 60 seconds per package.  

My First idea was to patch the deployment scripts. While looking at the Code on GitHub, i found it might be much more convenient to create updated aks base images and put them onto my AzureStack Marketplace. 

## The Current State

As of this Post, the "Official" Documented version of AKS-Engine for AzureStack Hub is v0.55.4.  
This Version has Built In Support for Kubernetes up to 1.17.11

Preseeded deployments requires the Following Marketplace Images to be deployed:  

{% highlight yaml %}
	AzureStackAKSUbuntu1604OSImageConfig = AzureOSImageConfig{
		ImageOffer:     "aks",
		ImageSku:       "aks-engine-ubuntu-1604-202007",
		ImagePublisher: "microsoft-aks",
		ImageVersion:   "2020.09.14",
	}
{% endhighlight %}		

The Images have the following K8S Version Pre-Seeded:
{% highlight yaml %}

K8S_VERSIONS="
1.17.11-azs
1.17.9-azs
1.16.14-azs
1.16.13-azs
1.15.12-azs
1.15.11-azs
"
{% endhighlight %}

However, while reading the Release Notes for AKS-Engine on GitHub, i found newer versions exist with newer Image Support:  

<figure class="full">
	<img src="/images/aks_release.png" alt="">
	<figcaption>AKS Images</figcaption>
</figure>

Yes, it says v0.60.0 would support up to K8S-AzS v1.18.15, but that is *NOT* in the current image.  

This would require image 2021.01.28, as per [chore: rev 2021.01.28](https://github.com/Azure/aks-engine/pull/4223/commits/976e0c41e75a4bfe24741f5dfec78b006ad6fdfc)


## Building a new Image

While reading and Browsing the [aks-engine GitHub releases](https://github.com/Azure/aks-engine/tree/release-v0.60.0), i found it would be a convenient way for me to create new microsoft-aks images using packer the way Microsoft does...    
The Packer building Process used my microsoft is essentially an automated approach used in their Pipelines to create new Azure Images.  
So why not just use this for AzureStackHub ( keep in mind, it's like Azure )

As you can see from the Changelog, aks-engine *has* newer support for AzureStack Hub, however, aks-engine would complain about a missing aks sku version.

To create a newer image offer, we will use packer and the artifacts from the aks-engine GitHub repository.  
But before we start, we will need to do some prerequisite.  

In a Nutshell, the Process will deploy an Ubuntu Base VM in Azure along with a temporary resource group, pull all supported docker Containers for aks from mcr an pre-seed all software versions. Some hardening and updates will run, and once finished and cleaned up, a new image will be provided along with the SAS Url to download.  

### Requirements
I run the imaging process from an Ubuntu 18.04 Machine. you can use a VM, WSL2 or any Linux host / Docker Image.

Install the following Packages:
- Packer from https://www.packer.io/downloads
- make utils ( sudo apt install make )
- git ( sudo apt install git)
- an Azure account ( the images will be created in Public Azure)

We also would need an Azure Service Principal in an Azure Subscription, as the build process runs in Azure.

### Creating a v0.60.0 Compliant image

First of all, use git to clone into the aks-engine repo  

{% highlight shell %}
git clone git@github.com:Azure/aks-engine.git
cd aks-engine
{% endhighlight %}
every release of aks sitś in a new bracnch
to get a list of all released versions of aks-engine, simply type  

{% highlight shell %}
git branch -r| grep release
{% endhighlight %}

as we want to create a v0.60.0 compliant image, we checkout the corresponding release branch:

{% highlight shell %}
git checkout release-v0.60.0
{% endhighlight %}


for the packer process, microsoft automatically creates a settings.json file. to pass required parameters for the file, we export some shell variables for the init process:  
(Replace ARM Parameters with your values)

{% highlight shell %}
export SUBSCRIPTION_ID=<ARM_SUBSCRIPTION_ID>
export TENANT_ID=<ARM_TENANT_ID>
export CLIENT_SECRET=<ARM_CLIENT_SECRET>
export CLIENT_ID=<ARM_CLIENT_ID>
export AZURE_RESOURCE_GROUP_NAME=packer ( an existing resource group ) 
export AZURE_VM_SIZE=Standard_F4
export AZURE_LOCATION=germanywestcentral
export UBUNTU_SKU=16.04
{% endhighlight %}

once those variables are set, we can run the initialization of the Environment.  

first, we connect programmatic to the azure environment with *make  az-login*  
{% highlight shell %}
make  az-login
{% endhighlight %}

if the login is successful, you SP and Credentials worked.

We can now use the *make init-packer* command to initialize the Packer Environment (e.g. create the storage account)  

{% highlight shell %}
make init-packer
{% endhighlight %}

<figure class="full">
	<img src="/images/packer_init.png" alt="">
	<figcaption>init packer</figcaption>
</figure>
*note: i skipped the az-login in the picture above as it shows credentials :-)   


once done with the init process, we can start to build our image.  

{% highlight shell %}
make build-packer
{% endhighlight %}
<figure class="full">
	<img src="/images/packer_build.png" alt="">
	<figcaption>build-packer</figcaption>
</figure>


Sit back and relax , the Process will take   
a good amount of time as a lot of stuff will be seeded into the image.



Once the Image has been created, you will be presented with URL's to download the image / template.  
We need the "OSDiskUriReadOnlySas" URL now, as we will use it to add the Image to the AzureStack.

<figure class="full">
	<img src="/images/packer_finished.png" alt="">
	<figcaption>packer-finished</figcaption>
</figure>

Review your Image SKU from /pkg/api/azenvtypes.go .   
For example, for an aks-engine-ubuntu-1804-202007 image:

{% highlight yaml %}
	AKSUbuntu1804OSImageConfig = AzureOSImageConfig{
		ImageOffer:     "aks",
		ImageSku:       "aks-engine-ubuntu-1804-202007",
		ImagePublisher: "microsoft-aks",
		ImageVersion:   "2021.01.28",
	}
{% endhighlight %}


### Uploading the Image to AzureStackHub
Goto you AzureStackHub AdminPortal. Click on Compute > VM Images > Add  to add a new Image.  
Fill in the Image Offer/SKU/Publisher/Vesrion for above, and paste the OSDiskUriReadOnlySas in "OS disk blob URi".  

<figure class="full">
	<img src="/images/add_vm_image.png" alt="">
	<figcaption>add vm image</figcaption>
</figure>


It will take a while for the Image Upload to be finished. Once the Image state changed fro creating to succeeded, you are ready to test a new deployment with the aks-engine version you used to build the image.  


Here are 2 Examples of AKS Nodes running different image sku's in different Clusters:

For an 16.04, engine v0.55.4 deployed Cluster running K8S v1.17.11:

<figure class="full">
	<img src="/images/aks_1604.png" alt="">
	<figcaption>add vm image</figcaption>
</figure>

For an 18.04, engine v0.60.0 deployed Cluster running K8S v1.18.15:
<figure class="full">
	<img src="/images/aks_1804.png" alt="">
	<figcaption>add vm image</figcaption>
</figure>


I am running a lot of different AKS Version in my lab with now issues.  
In the configs i am using, i run them with the azure cni and calico deployed.  
Happy k8s-ing
