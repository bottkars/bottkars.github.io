---
layout: post
title: "updated AKS Image SKU`s on AzureSTack Hub for new(er) Versions of AKS-Engine"
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

Howeve



<figure class="full">
	<img src="/images/aks_compute_images.png" alt="">
	<figcaption>AKS Images</figcaption>
</figure>


You know should have a Running azuredisk-csi-driver Environment on your AzureSTack Hub. 
Stay Tuned for Part 2 including DataProtection with PowerProtect Datamanager ...