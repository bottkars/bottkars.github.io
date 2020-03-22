---
layout: post
title: cf-for-k8s -> Run Cloudfoundry on Azurestack AKS
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
	<figcaption>set-pipeline</figcaption>
</figure>  

we should now have a Paused cf-for-k8s Pipeline in our UI :

<figure class="full">
	<img src="images/cf_for_k8s_pused_pipeline.png" alt="">
	<figcaption>paused pipeline</figcaption>
</figure>  

the Pipeline has the folllowing Task Flow:

- deploy-aks-cluster ( c.a. 12 Minutes)
- validate-aks-cluster ( sonobuoy basic validation, ca. 5 minutes)
- install kubeapps ( default in my clusters, bitnami catalog for HELM Repo´s)
- scale-aks-clusters ( for cf-for-k8s, we go to add some more nodes :-) )
- deploy-cf-for-k8s

While the first 4 Tasks are default for my AKS Deployments, we will focus on the *deploy-cf-for-k8s* task

## deploy-cf-for-k8s

the *deploy-cf-for-k8s*  requires the following resources from either github or local storage:
- azs-concourse (required tasks scripts)
- bosh-cli-release ( latest version of bosh cli)
- cf-for-k8s-master ( cf-for-k8smaster branch)
- yml2json-release ( yml to json converter)
- platform-automation-image (base image to run scripts)

## the tasks

## Monitoring the installation



cf-for-k8s will be deployed using [k14tools](https://k14s.io/) for an easy composable deployment.
the pipeline does that during the install

<figure class="full">
	<img src="images/kapp-deploy.png" alt="">
	<figcaption>get kubeconfig</figcaption>
</figure>

the pipeline may succeed,  with cf-for-k8s not finished deploying. the deployment time varies on multiple factors including internet speed. however, the kapp deployment may still be ongoing when the pipeline is finished.

to minitor the deployment on your machine, you can install [k14tools](https://k14s.io/) on your machine following the instructions on their site. 
kapp requires a kubeconfig file to access your cluster.
copy you kubeconfig file ( the deploy-aks task stores that on your s3 store after deployment)

i have an alias that copies my latest kubeconfig file:

```bash
get-kubeconfig
```
<figure class="full">
	<img src="images/get-kubeconfig.png" alt="">
	<figcaption>get kubeconfig</figcaption>
</figure>

to monitor / inspect the deployment, run

```bash
kapp inspect -a cf
```
<figure class="full">
	<img src="images/kapp-inspect.png" alt="">
	<figcaption>get kubeconfig</figcaption>
</figure>

in short, oince all pods in namespace cf-system are running, the system should be ready ( be aware, as there are daily changes, a deployment *might* fail)

[k9s](https://k9scli.io/) can give you a great overview of the running pods

<figure class="full">
	<img src="images/k9s-ready.png" alt="">
	<figcaption>get kubeconfig</figcaption>
</figure>

## connect to you cloudfoundry environment

cf-for-k8s depoloy´s a Service Type Loadbalancer per default.
the Pipeline create a dns a record for the cf domain you specified for the pipeline.

the admin´s password is autogenerated and stored in the cf-yalues.yml that get stored on your s3 location.
in my (direnv) environment, i receive the firl / the credentials   with

```bash
get-cfvalues
get-cfadmin
```

### connecting to cf api and logging in:

```bash
get-cfvalues
cf api api.cf.local.azurestack.external --skip-ssl-validation
cf auth admin $(get-cfadmin)
```

<figure class="full">
	<img src="images/cf-auth.png" alt="">
	<figcaption>cf auth</figcaption>
</figure>

there are no orgs and spaces defined per default, so we are going to create:
```bash
cf create-org demo
cf create-space test -o demo
cf target -o demo -s test
```

<figure class="full">
	<img src="/images/create-org.png" alt="">
	<figcaption>cf auth</figcaption>
</figure>

### push a docker container

to run docker containers in cloudfoundry, you also have to 
```bash
cf enable-feature-flag diego_docker
```

now it is time deploy our first docker container to cf-for-k8s

push the diego-docker-app from the cloudfoundry project on dockerhub

```bash
cf push diego-docker-app -o cloudfoundry/diego-docker-app
```

<figure class="full">
	<img src="images/diego-docker-app.png" alt="">
	<figcaption>cf auth</figcaption>
</figure>


we can now browse the endpoint og the demo app http://diego-docker-app.cf.local.azurestack.external/env

<figure class="full">
	<img src="/images/cf-diego-docker-browser.png" alt="">
	<figcaption>cf auth</figcaption>
</figure>

or use curl:

```
curl http://diego-docker-app.cf.local.azurestack.external/env
```


### pushing an app from source
cf-for-k8s utilizes cloudnative buildpacks using [kpack](https://github.com/pivotal/kpack)
In essence, a watcher it running in the kpack namespace to monitor new build request from cloudcontroller.
a "build pod" will run the build process of detecting, analyzing building and exportin the image

once a new request is detected, the clusterbuilder will create an image an ship it to the (gcr) registry.
from there

<figure class="full">
	<img src="/images/cf-build-pod.png" alt="">
	<figcaption>cf auth</figcaption>
</figure>

you can always view and monitor the image builder process by viewing the image and use the [log_tail](https://github.com/pivotal/kpack/blob/master/docs/logs.md) utility to view the builder logs:

```bash
kubectl get images --namespace cf-system
logs --image caf42222-dc42-45ce-b11e-7f81ae511e06 --namespace cf-system
```

<figure class="full">
	<img src="/images/image_and_log.png" alt="">
	<figcaption>cf auth</figcaption>
</figure>

