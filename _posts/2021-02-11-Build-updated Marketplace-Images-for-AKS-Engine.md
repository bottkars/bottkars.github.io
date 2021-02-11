---
layout: post
title: "update AKS Image SKU`s on AzureStack Hub to use new(er) Versions of AKS-Engine and Kubernetes"
description: "stay up to date"
modified: 2021-02-09
comments: true
published: false
tags: [aks-engine, json, bash, csi, aks, AzureStackHub]
image:
  path: /images/aks_compute_images.png
  feature: aks_compute_images.png
  credit: 
  creditlink: 
---
# Up to Date AKS Images on AzureStack Hub

Recently i tried to deploy newer versions of Kubernetes using AKS Engine to my AzureStack hub, but i failed for some odd reasons. That made me thinking about how do get newer Versions deployed to my AzureStack Hub using aks-engine.   


## The Problem

If you want to use your own Image, or try a newer AKS Engine release, the Deployment will try to download the missing Packages from https://kubernetesartifacts.azureedge.net/kubernetes/ and the according Docker Images from mcr.microsoft.com/oss/kubernetes/  

This Process is most likely to fail for slow Networks, as the download scripts have timeouts of 60 seconds per package.  

My First idea was to patch the deployment scripts. While looking at the Code on github, i found it might be much more convenient to create updated aks base images and put them onto my AzureStack Marketplace. 

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

However, while reading the Relesenotes for AKS-Engine on GitHub, i found newer versions exist with newer Image Support:  

<figure class="full">
	<img src="/images/aks_release.png" alt="">
	<figcaption>AKS Images</figcaption>
</figure>

Yes, it says v0.60.0 would support up to K8S-AzS v1.18.15, but that is *NOT* in the current image.  

This would require image 2021.01.28, as per [chore: rev 2021.01.28](https://github.com/Azure/aks-engine/pull/4223/commits/976e0c41e75a4bfe24741f5dfec78b006ad6fdfc)


## Building a new Image

While reading and Browsing the github, i found a convenient way for me to create new microsoft-aks images using packer, as also leveraged by Microsoft.

As you can see from the Changelog, it *has* newer support for AzureStack Hub, however, aks-engine would complain about a missing aks sku version.

To create a newer image offer, we will use packer and the artifacts from the aks-engine github repo.  
But before we start, we will need to do some prereqs.  

In a Nutshell, the Process will deploy an Ubuntu Base VM in Azure along with a temporary resource group, pull all supported docker Containers for aks from mcr an pre-seed all software versions. Some hardenings and updates will run, and once finished and cleaned up, a new image will be provided along with the SAS Url to download.  

### Pre Requirements
I run the imaging process from an ubuntu 18.04 Machine. you can use a VM, WSL2 or any linux host / Docker Image.

Install the following Packages:
- Packer from https://www.packer.io/downloads
- make utils ( sudo apt install make )
- git ( sudo apt install git)

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

first, we connect programmatically to the azure environment with *make  az-login*  
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




OSType: Linux
StorageAccountLocation: GermanyWestCentral
OSDiskUri: https://aksimages1613053325.blob.core.windows.net/system/Microsoft.Compute/Images/aksengine-vhds/aksengine-1613053325-osDisk.0deadc7d-68dc-47d6-ab8c-eb4748693fb1.vhd
OSDiskUriReadOnlySas: https://aksimages1613053325.blob.core.windows.net/system/Microsoft.Compute/Images/aksengine-vhds/aksengine-1613053325-osDisk.0deadc7d-68dc-47d6-ab8c-eb4748693fb1.vhd?se=2021-03-11T16%3A00%3A25Z&sig=XioyUPj7WS6VW3rz1xin5bAA6BUOpiF%2BRVlgujWbeEI%3D&sp=r&sr=b&sv=2018-03-28
TemplateUri: https://aksimages1613053325.blob.core.windows.net/system/Microsoft.Compute/Images/aksengine-vhds/aksengine-1613053325-vmTemplate.0deadc7d-68dc-47d6-ab8c-eb4748693fb1.json
TemplateUriReadOnlySas: https://aksimages1613053325.blob.core.windows.net/system/Microsoft.Compute/Images/aksengine-vhds/aksengine-1613053325-vmTemplate.0deadc7d-68dc-47d6-ab8c-eb4748693fb1.json?se=2021-03-11T16%3A00%3A25Z&sig=sVJqV8L63HM9zB5pEQdDW3Uyk7wLhNWUBaJfReJlkiA%3D&sp=r&sr=b&sv=2018-03-28
