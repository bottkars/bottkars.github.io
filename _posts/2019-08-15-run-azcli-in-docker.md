---
layout: post
title: docker run - running the az cli in a container for AzureStack
description: "There must be an easy way zo connect to AzureStack"
modified: 2019-08-15
comments: true
tags: [cert post]
image:
  path: https://azurecomcdn.azureedge.net/cvt-8163d731844e53b6d7ebd76103b6d0409c63cd671a1cea2acc86b45244d13a43/images/page/services/kubernetes-service/docker/docker-logo.svg
  feature: docker-logo.svg
  credit: 
  creditlink: 
---

# Use az-cli from Docker to connect to AzureStack

in this post I will explain how to use the microsoft/azure-cli container image programmatically to connect to AzureStack

The easiest way to start the AzureCLI Container interactively is by using

{% highlight scss %}
docker run -it microsoft/azure-cli:latest
{% endhighlight %}

<figure class="third">
	<img src="/images/azcli_docker.png" alt="">
	<figcaption>azcli from dockerdocker</figcaption>
</figure>

However, this would require to manually configure the Cloud Environment
