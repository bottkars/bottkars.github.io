---
layout: post
title: docker run - running the az cli in a container for AzureStack
description: "There must be an easy way zo connect to AzureStack"
modified: 2019-08-15
comments: true
tags: [cert post]
image:
  path: /images/docker_run.gif
  feature: docker_run.gif
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
	<figcaption>azcli from docker</figcaption>
</figure>

While this might be just enough to run some commands in Azure or AzureStack one time, it is not sufficient to scale Multiple Sessions or different Cloud Environments.

To have a more efficient way to

However, this would require to manually configure the Cloud Environment
