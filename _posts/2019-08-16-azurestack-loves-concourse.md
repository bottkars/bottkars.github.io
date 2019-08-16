---
layout: post
title: fly set-pipeline azurestack chapter 1
description: "The awesome way to automate AzureStack"
modified: 2019-08-16
comments: true
published: true
tags: [AzureStack, concourse, azure, cli]
image:
  path: /images/pipeline_pcf.gif
  feature: pipeline_pcf.gif
  credit: 
  creditlink: 
---

# Use [Concourse CI](https://concourse-ci.org/) to Automate AzureStack - Part 1

This Post will focus on getting a base Concourse System Setup on Docker and create and run basic Pipelines and Tasks on AzureStack

## What Concourse ?

Concourse is a CI/CD Pipeline that was developed with ease of use in Mind.
Other than in well-known CI/CD Tools, it *DOES NOT* install plugins or agents.
Concourse runs Tasks in OCI Compatible Containers.
All in and output´s of Jobs are resources, and are defined by a Type.
Based on the Resource Type, concourse will detect Version Changes of the resources. Concourse comes with a view built-in-types like *git* and *S3*, but you can very easy integrate you own Types. 

read more about Concourse at [Concourse-CI](https://concourse-ci.org/docs.html)

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
### run the container(s)
Once the file is downloaded, we start Concourse with
*docker-compose up* ( in attached mode )
that will load the required containers and start concourse with the web-instance running on 8080.
If you want to run on a different Port, check the docker-compose.yml

{% highlight scss %}
docker-compose up
{% endhighlight %}

<figure class="full">
	<img src="/images/concourse-up.gif" alt="">
	<figcaption>concourse up</figcaption>
</figure>

### Download the CLI
Now that we have concourse up and running, we download the *fly cli* commandline for concourse. therefore, we use the browser to browse to *http://localhost:8080*

<figure class="full">
	<img src="/images/concourse_fly_1.png" alt="">
	<figcaption>concourse ui</figcaption>
</figure>

Click on the Icon for your Operating System to Download the fly cli to your computer
Copy the fly cli into your path

### Connect to Concourse 
Open a new Shell ( Powershell, Bash ) to start with our first commands.

As we can target multiple instances of Concourse, we first need to target our instance and log in.

therefore, we use fly -t <<targetname>> login -c <url> -b

```bash
fly -t docker login -c http://localhost:8080 -b
```
<figure class="full">
	<img src="/images/concourse-login.png" alt="">
	<figcaption>concourse ui</figcaption>
</figure>

the -b switch will open a new Browser Window and point you to the login.
Login with user: test, password: test.

This should log you in to concourse at cli and Browser Interface.

## My First Pipeline

For our First pipeline, I created a template repository on Github.
go to [azurestack-concourse-tasks-template](https://github.com/bottkars/azurestack-concourse-tasks-template), and click on the *Use this template* button.

<figure class="full">
	<img src="/images/use-this-template.png" alt="">
	<figcaption>template</figcaption>
</figure>

Github will ask you for a Repository name, choose *azcli-concourse*.
Once the repository is created, clone into the repository from commandline, eg *git clone https://github.com/youruser/azcli-concourse.git*

<figure class="full">
	<img src="/images/clone-repo.png" alt="">
	<figcaption>repo</figcaption>
</figure>

cd into the directory you just cloned.

you will find yml files in that directory. open the parameter.yml file

{% highlight scss %}
azcli-concourse-uri: <your github repo>
{% endhighlight %}

replace the *<your github repo>* with your repo url.

have a look at 01-azcli-pipeline.yml

{% highlight scss %}
---
# example tasks
resources:
- name: azcli-concourse
  type: git
  icon: github-circle
  source: 
    uri: ((azcli-concourse-uri))
    branch: master

- name: az-cli-image
  icon: azure
  type: docker-image
  source: 
    repository: microsoft/azure-cli

jobs:
- name: test-azcli 
  plan:
  - get: acli-concourse
    trigger: true
  - get: az-cli-image
    trigger: true
  - task: test-azcli
    image: az-cli-image
    file: azcli-concourse/tasks/test-task.yml

---
{% endhighlight %}

this is our first pipeline. it has 2 resources configured:
- azcli-concourse, a git resource that hold´s our tasks
- az-cli-image, a docker image that contains the az cli

Now tha we have edited the Parameter to point to our github repo, we can load the pipeline into concourse

{% highlight scss %}
azcli-concourse-uri: <your github repo>
{% endhighlight %}

<figure class="full">
	<img src="/images/set-pipeline.png" alt="">
	<figcaption>set-pipeline</figcaption>
</figure>

Apply the Pipeline configuration by confirming with *y*

No go Back to the Web Interface. You should now see the pipeline named azurestack in a paused state

<figure class="full">
	<img src="/images/pipe-1.png" alt="">
	<figcaption>set-pipeline</figcaption>
</figure>

Hover over the Blue Boy and click on the name azurestack.
this should open the pipeline view.

<figure class="full">
	<img src="/images/pipeview1.png" alt="">
	<figcaption>pipeview</figcaption>
</figure>

You can see the 2 resources az-cli-image and azcli-concourse

if you click on each of them, you will notice they have been checked successfully

<figure class="full">
	<img src="/images/checked.png" alt="">
	<figcaption>checked resource</figcaption>
</figure>

concourse will automatically check for new version of the resources  ( git fetch, docker, s3 ..), and this could trigger a build in a Pipeline.

Now, it is time to unpause our Pipeline. In  the web-interface, click on the start button. Then click on the test-azcli job and manually trigger the build by pushing the + button.

this will trigger you build....

<figure class="full">
	<img src="/images/first-build.gif" alt="">
	<figcaption>first build</figcaption>
</figure>