---
layout: post
title: docker run - running the az cli in a container for AzureStack
description: "There must be an easy way zo connect to AzureStack"
modified: 2019-08-15
comments: true
tags: [cert post]
image:
  path: /images/azcli_docker.png
  feature: azcli_docker.png
  credit: 
  creditlink: 
---

# Use az-cli from Docker to connect to AzureStack

in this post I will explain how to use the microsoft/azure-cli container image programmatically to connect to AzureStack

<figure class="third">
	<img src="/images/azcli_docker.png alt=">
	<figcaption>azcli from dockerdocker</figcaption>
</figure>

{% highlight scss %}
openssl.exe x509 -inform DER  -outform pem -in .\root.cer -out root.pem
{% endhighlight %}