---
layout: post
title: "updated AKS Image SKU`s on AzureSTack Hub for new(er) Versions of AKS-Engine"
description: "stay up to date"
modified: 2021-02-09
comments: true
published: no
tags: [aks-engine, json, bash, csi, aks, AzureStackHub]
image:
  path: /images/aks_compute_images.png
  feature: aks_compute_images.png
  credit: 
  creditlink: 
---
# Up to Date AKS Images on AzureSTack Hub

Recently i tried to deploy newer versions of Kubernetes using AKS Engine to my AzureStack hub, but i failed for some odd reasons. That made me thinking about how do get newer Versions deployed to my AzureStack Hub using aks-engine.   


## The Problem

If you want to use your own Image, or try a newer AKS Engine release, the Deployment will try to download the missing Packages from https://kubernetesartifacts.azureedge.net/kubernetes/ and the according Docker Images from mcr.microsoft.com/oss/kubernetes/  

This Process is most likely to fail for slow Networks, as the download scripts have timouts of 60 seconds per package.  

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

Yes, it say v0.60.0 would support up to K8S-AzS v1.18.15, but that is *NOT* in the current image.  

This would require image 2021.01.28, as per [chore: rev 2021.01.28](https://github.com/Azure/aks-engine/pull/4223/commits/976e0c41e75a4bfe24741f5dfec78b006ad6fdfc)


## Building a new Image

While reading and Browsing the github, i found a convenient way for me to create new microsoft-aks images using packer, as also leveraged by Microsoft.

As you can see from the Changelog, it *has* newer support for AzureStack Hub, however, aks-engine would complain about a missing aks sku version.

To create a newer image offer, we will use packer and the artifacts from the aks-engine github repo.  
But before we start, we will need to do some prereqs.  

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


