---
layout: post
title: cf-for-k8s -> aRun Cloudfoundry on Azurestack AKS
description: "the way software get´s created in the cloud"
modified: 2019-08-24
comments: true
published: false
tags: [AzureStack, azure, concourse, cli, fly, cf-for-k8s, kubernetes, cloudfoundry]
image:
  path: /images/basic_azure.png
  feature: basic_azure.png
  credit: 
  creditlink: 
---

# cf-for-8 Pipeline for concourse-ci [Concourse CI](https://concourse-ci.org/) 

Before getting started:
cf-for-k8s installation is pretty straight forward. In this example i am using concourse-fi [Concourse CI](https://concourse-ci.org/), and the dployment scripts are custom tasks based out of my github repo [azs-concourse](https://github.com/bottkars/azs-concourse/tree/tanzu), where the pipeline used is [platform automation](https://raw.githubusercontent.com/bottkars/platform-automation/tanzu/pipeline_azurestack_aksengine.yml)

So we actually start with setting the Pipeline:

```bash
flyme set-pipeline -c ${AKS_PIPELINE}  -l ${PLATFORM_VARS} -l ${AKS_VARS} -p ${AKS_CLUSTER} -v cf_k8s_domain=cf.local.azurestack.external
```
in the above call, the following aliases / variables are used:

*flyme*:  is an alias for fly --target
*AKS_PIPELINE*: is the pipeline file
*PLATFORM_VARS*: Variables containing essential, pipeline independant Environment Variable, e.G. AzureSTack Endpoints ( leading with AZURE_) and general var´s
*AKS_VARS*: essntially, vars to control the AKS Engine ( cluster, size etc.. )
Example AKS_VARS:

```yaml
azs_concourse_branch: tanzu
aks:
    team: aks
    bucket: aks
    resource_group: cf-for-k8s
    orchestrator_release: 1.15
    orchestrator_version: 1.15.4
    orchestrator_version_update: 1.16.1
    engine_tagfilter: "v0.43.([1])"
    master:
      dns_prefix: cf-for-k8s
      vmsize: Standard_D2_v2
      node_count: 3 # 1, 3 or 5, at least 3 for upgradeable
      distro: aks-ubuntu-16.04
    agent:
      0:
        vmsize: Standard_D3_v2
        node_count: 3
        distro: aks-ubuntu-16.04
        new_node_count: 6
        pool_name: linuxpool
        ostype: Linux
    ssh_public_key: ssh-rsa AAAAB... 
  ```
<figure class="full">
	<img src="images/cf_for_k8s_set_pipeline.png" alt="">
	<figcaption>get-vms-rg</figcaption>
</figure>  


cf_for_k8s_set_pipeline


## Tasks and Anchors

First of all, we copy last weeks 04-azcli-pipeline.yml into 05-azcli-pipeline.yml


```yml
  RESOURCE_GROUP:
```

right under AZURE_CA_PATH.

In the run Part, right under *az account set --subscription ${AZURE_SUBSCRIPTION_ID}*, add the following code:

```bash
 az vm list --resource-group ${RESOURCE_GROUP} --output table
 ```

you new Task File should look like this now:

```yml
---
# this a task to get vm´s of a certain resource croup
platform: linux

params:
  PROFILE:
  CLOUD:
  # AzureStack AzureCloud AzureChinaCloud AzureUSGovernment AzureGermanCloud
  CA_CERT:
  ENDPOINT_RESOURCE_MANAGER:
  VAULT_DNS:
  SUFFIX_STORAGE_ENDPOINT:
  AZURE_TENANT_ID:
  AZURE_CLIENT_ID:
  AZURE_CLIENT_SECRET:
  AZURE_SUBSCRIPTION_ID:
  AZURE_CLI_CA_PATH:
  RESOURCE_GROUP:

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
     # --allow-no-subscriptions
    set -eux
    az account set --subscription ${AZURE_SUBSCRIPTION_ID}
    az vm list --resource-group ${RESOURCE_GROUP} --output table
```

Commit your changes

```bash
git add tasks/get-vms-rg.yml
git commit -a -m "added get-vms-rg"
git push
```

### adding the task to our pipeline and create anchors

The call of the task from the pipeline will essentially look like our basic task, just we have to add a parameter and change the name of the taskfile.
as this will create a lot of overhead in the Parameters, we create a YAML anchor for our "Standard"  Parameters of the task.

At the beginning of our 04-azcli-pipeline.yml, create the following Anchor:
(it should name your environment, in my case, the asdk, so azurestack_asdk_env)

```yml
azurestack_asdk_env: &azurestack_asdk_env
  CLOUD: ((asdk.cloud))
  CA_CERT: ((asdk.ca_cert))
  PROFILE: ((asdk.profile))
  ENDPOINT_RESOURCE_MANAGER: ((asdk.endpoint_resource_manager))
  VAULT_DNS:  ((asdk.vault_dns))
  SUFFIX_STORAGE_ENDPOINT: ((asdk.suffix_storage_endpoint))
  AZURE_TENANT_ID: ((asdk.tenant_id))
  AZURE_CLIENT_ID: ((asdk.client_id))
  AZURE_CLIENT_SECRET: ((asdk.client_secret))
  AZURE_SUBSCRIPTION_ID: ((asdk.subscription_id))
  AZURE_CLI_CA_PATH: "/usr/local/lib/python3.6/site-packages/certifi/cacert.pem"
```  

in our existing Task fo AzureSTack, we replace the parameters in the params section with

```yml
params:
 <<: *azurestack_asdk_env
```

now mark and copy the the basic task of your pipeline file.

insert it as a new task.
change the task name to get-vms-rg, and change the path of the task file to get-vms-rg.yml
Add a Parameter for the Resource Group, in my case asdk.resource group
the new task should look like this:
*(note, i am using the parameters with prefix asdk. in this example as this is my set of specific parameters for my asdk)*

```yml
- name: get-vms-rg
  plan:
  - get: azcli-concourse
    trigger: true
  - get: az-cli-image
    trigger: true
  - task: get-vms-rg
    image: az-cli-image
    file: azcli-concourse/tasks/get-vms-rg.yml
    params:
      <<: *azurestack_asdk_env
      RESOURCE_GROUP: ((asdk.resource_group))
```

The Anchor will instruct fly to insert the Section from the Anchor definition
edit the Parameter file to include the resource_group parameters:

```yml
asdk:
  tenant_id: "your tenant id"
  client_id: "your client id"
  client_secret: "your very secret secret"
  subscription_id: "your subscription id"
  endpoint_resource_manager: "https://management.local.azurestack.external"
  vault_dns: ".vault.local.azurestack.external"
  suffix_storage_endpoint: "local.azurestack.external"
  cloud: AzureStackUser
  profile: "2019-03-01-hybrid"
  azure_cli_ca_path: "/usr/local/lib/python3.6/site-packages/certifi/cacert.pem"
  ca_cert: |
    -----BEGIN CERTIFICATE-----
    <<you root ca>>
    -----END CERTIFICATE-----
  resource_group: "you resource group"
```

save the files

### Load the updated pipeline

we load Version 4 of our Pipeline now with

{% highlight scss %}
fly -t docker set-pipeline -p azurestack  -c 04-azcli-pipeline.yml -l parameters.yml 
{% endhighlight %}

<figure class="full">
	<img src="/images/get-vms-rg.png" alt="">
	<figcaption>get-vms-rg</figcaption>
</figure>

You may now create / apply anchors and tasks for your different Azure/AzureStack Environments

## Do I need to write a task file each and every time ?

No, you do not have. Originally, the run Part of the task was Part of the Pipeline as well.
And for testing Purposes, i would even recommend to create a short test-pipeline for you task including the run statement.
That would allow you for easier testing and scripting WITHOUT applying changes to your master pipeline.

### Example

create a pipeline file called *script-test.yml* .

Put in your Anchor(s).
We do not need resource definitions, as we even call the image to use from within the Task.
We to not trigger the job, as we want to run it manually.

This is a basic task i user for script testing. Modify the run section to your needs.

```yml
---
# script developement pipeline
azurestack_asdk_env: &azurestack_asdk_env
  CLOUD: ((asdk.cloud))
  CA_CERT: ((asdk.ca_cert))
  PROFILE: ((asdk.profile))
  ENDPOINT_RESOURCE_MANAGER: ((asdk.endpoint_resource_manager))
  VAULT_DNS:  ((asdk.vault_dns))
  SUFFIX_STORAGE_ENDPOINT: ((asdk.suffix_storage_endpoint))
  AZURE_TENANT_ID: ((asdk.tenant_id))
  AZURE_CLIENT_ID: ((asdk.client_id))
  AZURE_CLIENT_SECRET: ((asdk.client_secret))
  AZURE_SUBSCRIPTION_ID: ((asdk.subscription_id))
  AZURE_CLI_CA_PATH: "/usr/local/lib/python3.6/site-packages/certifi/cacert.pem"

jobs:

- name: script-test
  plan:
  - task: script-test
    config:
      platform: linux
      params:
        <<: *azurestack_asdk_env
        RESOURCE_GROUP: ((asdk.resource_group))
      image_resource:
        type: docker-image
        source: {repository: microsoft/azure-cli}
      outputs:
      - name: result
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
              # --allow-no-subscriptions
            set -eux
            RESULT=$(az vm list --output json)
            echo $RESULT
            echo $RESULT > ./result/result.json
```

Now start a new pipeline called *script-test* with the new Pipeline file

```bash
fly -t docker sp -p script-test -c .\script-test.yml -l .\parameters.yml
```

The new Pipeline should be in a paused mode. Press the Play Button to start

<figure class="full">
	<img src="/images/paused-script-pipe.png" alt="">
	<figcaption>paused-script-pipe</figcaption>
</figure>

when you click in the script-test pipeline, you will see only one job, no dependencies, no triggers.
Trigger a build by clicking the plus button

<figure class="full">
	<img src="/images/script-test-trigger.png" alt="">
	<figcaption>script-test-trigger</figcaption>
</figure>

This should run your script.
You can see from the pipeline file that inline Scripting makes you pipeline quite large.  
My preferred method is to put the scripts in task files and load them from GitHub.  
You even can have versioned scripts zipped on external resources.  
That will also allow to trigger a new build on script change.

We will dive into that in one of the next Chapters.

For now, familiarize yourself with Anchors, internal and external tasks, and even have a look at the fly cli for method´s to pass tasks from directories

