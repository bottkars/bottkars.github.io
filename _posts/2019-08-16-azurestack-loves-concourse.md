---
layout: post
title: fly set-pipeline azurestack
description: "The awesome way to automate AzureStack"
modified: 2019-08-16
comments: true
published: false
tags: [AzureStack, concourse,]
image:
  path: /images/pipeline_pcf.gif
  feature: pipeline_pcf.gif
  credit: 
  creditlink: 
---

# Use [Concourse CI](https://concourse-ci.org/) to Automate AzureStack

This Post will focus on getting a base Concourse System Setup on Docker and create and run basic Pipelines and Tasks on AzureStack

## First things first

Create a Directory of your choice in my case Workshop, and cd into it
Before we start with our first pipeline, we need to get concourse up and running.
The easiest way to get started with concourse is using docker.
Concourse-CI provides a generic docker-compose file that will fit our needs for that course.

### download the file

linux/OSX users simply enter

```bash
wget https://concourse-ci.org/docker-compose.yml
```

if you are running Windows, set docker desktop to linux Containers and run

```Powershell
Invoke-Webrequest https://concourse-ci.org/docker-compose.yml -OutFile docker-compose.yml
```

Once the file is downloaded, we start Concourse with
*docker-compose up* ( in attached mode )

{% highlight scss %}
docker-compose up
{% endhighlight %}

<figure class="half">
	<img src="/images/concourse-up.gif" alt="">
	<figcaption>concourse up</figcaption>
</figure>
