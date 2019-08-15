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

## the basics

The easiest way to start the AzureCLI Container interactively is by using

{% highlight scss %}
docker run -it microsoft/azure-cli:latest
{% endhighlight %}

<figure class="full">
	<img src="/images/azcli_docker.png" alt="">
	<figcaption>azcli from docker</figcaption>
</figure>

## the Idea
While this might be just enough to run some commands in Azure or AzureStack one time, it is not sufficient to scale Multiple Sessions or different Cloud Environments.

So we need to have a more efficient way to run the Container.
One way would be passing Environment Variables o the Container, but I was looking for a more flexible approach.

The idea here is to use docker volumes to mount local directories into the docker container.

By leveraging *docker run -it -v <<volume>>:/path*, we should be able to pass environments, variables, files  and scripts to the container.
Example:

{% highlight scss %}
WORKSPACE=wokspace
docker run -it --rm \
    -v $(pwd)/vars:/${WORKSPACE}/vars \
    -v $(pwd)/scripts:/${WORKSPACE}/scripts \
    -v $(pwd)/certs:/${WORKSPACE}/certs \
    -w /${WORKSPACE} microsoft/azure-cli
{% endhighlight %}

to do so, i create 3 Directories:
- certs, contains the Azure Stack root ca 
- vars, contains environment specific vars
- scripts, contains the startup script for the azure env

## the vars directory file
the vars directory will hold
- *.env.sh*
- *.secrets*
a typical env.sh file would contain:

{% highlight scss %}
AZURE_CLI_CA_PATH="/usr/local/lib/python3.6/site-packages/certifi/cacert.pem"
PROFILE="2019-03-01-hybrid"
CA_CERT=root.pem
ENDPOINT_RESOURCE_MANAGER="https://management.local.azurestack.external"
VAULT_DNS=".vault.local.azurestack.external"
SUFFIX_STORAGE_ENDPOINT="local.azurestack.external"
AZURE_TENANT_ID=""
AZURE_SUBSCRIPTION_ID=""
{% endhighlight %}

the *.secrets* file is and option, and  will hold and Azure Service Principle to login programmatically.

it contains:

{% highlight scss %}
#!/bin/bash
export AZURE_CLIENT_ID=""
export AZURE_CLIENT_SECRET=""
{% endhighlight %}

If you do not want to expose the secrets in a file, you may pass them ass environment Variables.

## the scripts directory

the Script Directory in essence host the start script that you will execute from within the Container

it will 
- append the root ca cert to the az cli certificates
- Create the Cloud Environment for you AzureStack
- Signs in to AZS with the Service Principal, if provided

{% highlight scss %}
!/bin/bash
pushd $(pwd)
cd "$(dirname "$0")"
source ../vars/.secrets
set -eux
source ../vars/.env.sh
if [ -z "${CA_CERT}" ]
then
    echo "no custom root ca found"
else
    cat ../certs/${CA_CERT} >> ${AZURE_CLI_CA_PATH} 
fi

az cloud register -n AzureStackUser \
--endpoint-resource-manager ${ENDPOINT_RESOURCE_MANAGER} \
--suffix-storage-endpoint ${SUFFIX_STORAGE_ENDPOINT} \
--suffix-keyvault-dns ${VAULT_DNS} \
--profile ${PROFILE}
az cloud set -n AzureStackUser
set +eux
if [ -z "${AZURE_CLIENT_ID}" ] || [ -z "${AZURE_CLIENT_SECRET}"  ]
then
    echo "no Client Credentials found, skipping login"
else
    az login --service-principal \
    -u ${AZURE_CLIENT_ID} \
    -p ${AZURE_CLIENT_SECRET} \
    --tenant ${AZURE_TENANT_ID}  
    az account set --subscription ${AZURE_SUBSCRIPTION_ID}
fi
{% endhighlight %}

## putting it all together

with the above files in place, we would start docker with

```bash
WORKSPACE=workspace
docker run -it --rm \
    -v $(pwd)/vars:/${WORKSPACE}/vars \
    -v $(pwd)/scripts:/${WORKSPACE}/scripts \
    -v $(pwd)/certs:/${WORKSPACE}/certs \
    -w /${WORKSPACE} microsoft/azure-cli
```

<figure class="full">
	<img src="/images/docker_azcli_connect.png" alt="">
	<figcaption>azcli from docker</figcaption>
</figure>

once in, we can start our environment and connect to our AzureStack endpoint:

