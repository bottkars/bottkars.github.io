---
layout: post
title: fly set-hybrid -> automation for azure and azurestack chapter 2
description: "The awesome way to automate Azure and AzureStack"
modified: 2019-08-17
comments: true
published: false
tags: [AzureStack, concourse, azure, cli, Azure]
image:
  path: /images/pipeline_pcf.gif
  feature: pipeline_pcf.gif
  credit: 
  creditlink: 
---

# Use [Concourse CI](https://concourse-ci.org/) to automate Azure and AzureStack - Chapter 2

This Chapter will we will create out firstTask that let us

- connect to the Cloud (Azure/AzureStack )
- template a task
- create Jobs

## creating a new Task

in the Previous chapter we use the test-task that i Provided in the github template. From now on, we are writing the Tasks on our own ...

let us start with the "base" task, that basically will test if we can connect to the cloud (s).
in [this post](_posts/2019-08-15-run-azcli-in-docker.md) i explained how to use azcli for AzureSTack with Docker. we will basically meme the same for our basic task
From now on i recommend using Visual Studio Code for all edit´s
You might want to install the [Concourse Pipeline Extension](https://marketplace.visualstudio.com/items?itemName=Pivotal.vscode-concourse)

## the basic task

create a new file ./tasks/basic-task.yml

ad the following lines to the file:

{% highlight scss %}
platform: linux

params.
  PROFILE:
  CLOUD:
  CA_CERT:
  ENDPOINT_RESOURCE_MANAGER:
  VAULT_DNS:
  SUFFIX_STORAGE_ENDPOINT:
  AZURE_TENANT_ID:
  AZURE_CLIENT_ID:
  AZURE_CLIENT_SECRET:
  AZURE_SUBSCRIPTION_ID:
  AZURE_CLI_CA_PATH: "/usr/local/lib/python3.6/site-packages/certifi/cacert.pem"
{% endhighlight %}

the *platform* parameter identifies the platform stack ( worker type ) to run on.  

The *parameters* section contains the (possible) Parameters we can provide to our task. We will provide the Parameters later from our Pipeline
As we are going to define a Custom Cloud Profile in case of AzureStack, this will also define our custom endpoints

Next, we add a "run" section. The run section is essence is the Script to be executed. if the script exits with a failure code, the Build Task will considered failed.

add the following lines to the task file:

{% highlight scss %}
run:
  path: bash
  args:
  - "-c"
  - |
    set -eux
    case ${CLOUD} in

    AzureStackUser)
        if [[ -z "${CA_CERT}" ]]
        then
            echo "no Custom root ca cert provided"
        else
            echo "${CA_CERT}" >> ${AZURE_CLI_CA_PATH}
        fi
        az cloud register -n ${CLOUD} \
        --endpoint-resource-manager ${ENDPOINT_RESOURCE_MANAGER} \
        --suffix-storage-endpoint ${SUFFIX_STORAGE_ENDPOINT} \
        --suffix-keyvault-dns ${VAULT_DNS} \
        --profile ${PROFILE}
        ;;

    *)
        echo "Nothing to do here"
        ;;
    esac

    az cloud set -n ${CLOUD}
    az cloud list --output table
    set +x
    az login --service-principal \
     -u ${AZURE_CLIENT_ID} \
     -p ${AZURE_CLIENT_SECRET} \
     --tenant ${AZURE_TENANT_ID}
    set -eux
    az account set --subscription ${AZURE_SUBSCRIPTION_ID}
{% endhighlight %}

This will evaluate the cloud type and load the appropriate Profile. For Azure Stack, we create a Cloud Profile with the endpoints passed from the Parameters.

## Going Secure from now

Before we edit our parameter file, it is time to go secure from now.

1. create a ssh key for your Pipeline Repository
2. Set the repository to private
3. add the ssh key to Deploy Keys
4. set the ssh key for the github resource

1. to create an ssh key for your Pipeline Repository, run

{% highlight scss %}
ssh-keygen -t rsa -b 4096 -C mypipeline@github.com -f ~/.ssh/azcli_demo_key -N ""
{% endhighlight %}

2. Set the repository to private
Growse to your Github Repository. Go to the settings in the upper right.

<figure class="full">
	<img src="/images/git_settings.png" alt="">
	<figcaption>git settings</figcaption>
</figure>

scroll down to the "Danger Zone" and click on make Private

<figure class="full">
	<img src="/images/danger_zone.png" alt="">
	<figcaption>git settings</figcaption>
</figure>

3. Add the Deploy key
Go to the Deploy key Section on the right

<figure class="full">
	<img src="/images/deploy_key.png" alt="">
	<figcaption>git settings</figcaption>
</figure>

Click on Add Key to add you ssh public key created in step 1.
Insert you Key, Check allow Write access and click Add Key

We will change the Pipeline Git Resource later accordingly.

## commit the current files