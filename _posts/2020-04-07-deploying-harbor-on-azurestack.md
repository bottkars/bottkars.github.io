---
layout: post
title: deploy and run Harbor COntainer Registry as VM on Azurestack for AKS
description: "the easiest way to get a secure container registry on Azurestack"
modified: 2020-04-07
comments: true
published: false
tags: [AzureStack, azure, harbor, kubernetes, cloudfoundry]
image:
  path: /images/k9s-ready.png
  feature: k9s-ready.png
  credit: 
  creditlink: 
---

# arm template for Harbor Contaimer Registry [Project Harbor ](https://goharbor.io/) 

this is an explanation of my ARM template for Harbor Container Registry. For details on Harbor head over to  [Project Harbor ](https://goharbor.io/).

The Template will deploy an Ubuntu 18.04 VM with Docker Engine and Harbor the official from GitHub Repo.
You can opt to have selfsigned certificates created automatically for you OR use custom Certificates form you CA.


Before we start the deployment, wi need to check that we have
 - a ssh Public Key in Place ( i default to ~/.ssh/id_rsa.pub in my code samples)
 - connection to AzureSTack from AZ CLI
 - Ubuntu 18.04 LTS Marketplace Image on Azurestack
 - Custom Script Extension for Linux on Azurestack
 - internet connection to dockerhub, canonical repo´s and GitHub

in the following examples, i deploy 2 Registry, 1 called devregistry with self-signed Certificates, and one called registry, to become my Production Registry using let´s engrypt Certificates

## Testing Deployment and Parameters
first we need to a variable before we start or Test the Deployment.
The Variable *DNS_LABEL_PREFIX* marks the hostname for the VM and will be registered with Azurestack´s DNS, eg
DNS_LABEL_PREFIX.location.cloudapp.dnsdomain
```bash
DNS_LABEL_PREFIX=devregistry # this should be the azurestack cloudapp dns name , e.g. Harbor, Mandatory
```
The name will also be used in the Generated Certificate for Self Signed Certs

If you are deploying using you own Certificates, you will also have provide you external hostname you created your Certificate for:

```bash
EXTERNAL_HOSTNAME=registry.home.labbuildr.com #external dns name
```

you can validate you deployment with:

### for Self Signed
```bash
DNS_LABEL_PREFIX=devregistry # this should be the azurestack cloudapp dns name , e.g. Harbor, Mandatory
az group create --name ${DNS_LABEL_PREFIX:?variable is empty} --location local
az deployment group validate --resource-group ${DNS_LABEL_PREFIX:?variable is empty} \
    --template-uri "https://raw.githubusercontent.com/bottkars/201-azurestack-harbor-registry/master/azuredeploy.json" \
    --parameters \
    sshKeyData="$(cat ~/.ssh/id_rsa.pub)" \
    HostDNSLabelPrefix=${DNS_LABEL_PREFIX:?variable is empty}
```
<figure class="full">
	<img src="/images/validate_devregistry.png" alt="">
	<figcaption>validate_devregistry</figcaption>
</figure>  

### for Provided Cerificates
for user provided Certificate, you also need to provide your
- hostCert, the Cerificate content of you Host or Domain Wildcard Cert
- certKey, the content of the matching Key for the above Certificate
  and, if your registry is not one of the Mozilla trusted registries,
- caCert, the Certificate content of you root ca for the docker engine
Un my Example, i use Let´s enrypt acme Certs and pass them via bash *cat* inline. Make sure to use Hyphens as the Certificates are Multiline values:
 

```bash
DNS_LABEL_PREFIX=registry #dns host label prefix 
EXTERNAL_HOSTNAME=registry.home.labbuildr.com #external dns name
az group create --name ${DNS_LABEL_PREFIX:?variable is empty} --location local
az deployment group validate --resource-group ${DNS_LABEL_PREFIX:?variable is empty}\
    --template-uri "https://raw.githubusercontent.com/bottkars/201-azurestack-harbor-registry/master/azuredeploy.json" \
    --parameters \
    sshKeyData="$(cat ~/.ssh/id_rsa.pub)" \
    HostDNSLabelPrefix=${DNS_LABEL_PREFIX:?variable is empty} \
    caCert="$(cat ~/workspace/.acme.sh/home.labbuildr.com/ca.cer)" \
    hostCert="$(cat ~/workspace/.acme.sh/home.labbuildr.com/home.labbuildr.com.cer)" \
    certKey="$(cat ~/workspace/.acme.sh/home.labbuildr.com/home.labbuildr.com.key)" \
    externalHostname=${EXTERNAL_HOSTNAME:?variable is empty}
```
<figure class="full">
	<img src="/images/validate_registry.png" alt="">
	<figcaption>validate_registry</figcaption>
</figure>  

If there are no errors from above commands, we should be ready to start the deployment

<figure class="full">
	<img src="/images/cf_for_k8s_paused_pipeline.png" alt="">
	<figcaption>paused pipeline</figcaption>
</figure>  

the Pipeline has the folllowing Task Flow:

- deploy-aks-cluster ( c.a. 12 Minutes)
- validate-aks-cluster ( sonobuoy basic validation, ca. 5 minutes)
- install kubeapps ( default in my clusters, bitnami catalog for HELM Repo´s)
- scale-aks-clusters ( for cf-for-k8s, we go to add some more nodes :-) )
- deploy-cf-for-k8s

While the first 4 Tasks are default for my AKS Deployments, we will focus on the *deploy-cf-for-k8s* task
( *note* i will write about the AKS Engine Pipeline soon !)

## deploy-cf-for-k8s

the [deploy-cf-for-k8s](https://github.com/bottkars/azs-concourse/blob/de92a34508a6d068c0d192fa18f60baf85ed3ff8/ci/scripts/deploy-cf-for-k8s.sh#L1-L70) task  requires the following resources from either github or local storage:
- azs-concourse (required tasks scripts)
- bosh-cli-release ( latest version of bosh cli)
- cf-for-k8s-master ( cf-for-k8smaster branch)
- yml2json-release ( yml to json converter)
- platform-automation-image (base image to run scripts)

also, the following variables need to be passed:

```yaml
  <<: *azure_env # youre azure-stack enfironment 
  DNS_DOMAIN: ((cf_k8s_domain)) # the cf domain
  GCR_CRED: ((gcr_cred)) # credentials for gcr
```     
where GCR_CRED contains the credentials to you Google Container Registry. You can provide them either form a secure store like credhub ( preferred way ), therefore simply load the credtials JSON file obtained from creating the secret with:

```bash
credhub set -n /concourse/<main or team>/gcr_cred -t json -v "$(cat ../aks/your-project-storage-creds.json)"
```

or load the variable to the pipeline from a YAML file in this example, gcr.yaml):

```yml
gcr_cred:
  type: service_account
  project_id: your-project_id
  private_key_id: your-private_key_id
  private_key: your-private_key
  client_email: your-client_email
  client_id: your-client_id
  auth_uri: your-auth_uri
  token_uri: your-token_uri
  auth_provider_x509_cert_url: your-auth_provider_x509_cert_url
  client_x509_cert_url: your-auth_uri
```  
and then 

```bash
fly -t concourse_target set-pipeline -c ${AKS_PIPELINE} \
  -l ${PLATFORM_VARS} \
  -l gcr.yml \
  -l ${AKS_VARS} \
  -p ${AKS_CLUSTER} \
  -v cf_k8s_domain=cf.local.azurestack.external
```


## the tasks
tbd
## Monitoring the installation



cf-for-k8s will be deployed using [k14tools](https://k14s.io/) for an easy composable deployment.
the pipeline does that during the install

<figure class="full">
	<img src="/images/kapp-deploy.png" alt="">
	<figcaption>kapp-deploy</figcaption>
</figure>

the pipeline may succeed,  with cf-for-k8s not finished deploying. the deployment time varies on multiple factors including internet speed. however, the kapp deployment may still be ongoing when the pipeline is finished.

to monitor the deployment on your machine, you can install [k14tools](https://k14s.io/) on your machine following the instructions on their site. 
kapp requires a kubeconfig file to access your cluster.
copy you kubeconfig file ( the deploy-aks task stores that on your s3 store after deployment)

i have an alias that copies my latest kubeconfig file:

```bash
get-kubeconfig
```
<figure class="full">
	<img src="/images/get-kubeconfig.png" alt="">
	<figcaption>get-kubeconfig</figcaption>
</figure>

to monitor / inspect the deployment, run

```bash
kapp inspect -a cf
```
<figure class="full">
	<img src="/images/kapp-inspect.png" alt="">
	<figcaption>kapp-inspect</figcaption>
</figure>

in short, oince all pods in namespace cf-system are running, the system should be ready ( be aware, as there are daily changes, a deployment *might* fail)

[k9s](https://k9scli.io/) can give you a great overview of the running pods

<figure class="full">
	<img src="/images/k9s-ready.png" alt="">
	<figcaption>get-kubeconfig</figcaption>
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

### connecting to cf api and logging in

```bash
get-cfvalues
cf api api.cf.local.azurestack.external --skip-ssl-validation
cf auth admin $(get-cfadmin)
```

<figure class="full">
	<img src="/images/cf-auth.png" alt="">
	<figcaption>cf_auth</figcaption>
</figure>

there are no orgs and spaces defined per default, so we are going to create:
```bash
cf create-org demo
cf create-space test -o demo
cf target -o demo -s test
```

<figure class="full">
	<img src="/images/create-org.png" alt="">
	<figcaption>cf-create-org</figcaption>
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
	<img src="/images/diego-docker-app.png" alt="">
	<figcaption>cf-diego-docker</figcaption>
</figure>


we can now browse the endpoint og the demo app http://diego-docker-app.cf.local.azurestack.external/env

<figure class="full">
	<img src="/images/cf-diego-docker-browser.png" alt="">
	<figcaption>cf-browser</figcaption>
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
	<figcaption>cf-build-pod</figcaption>
</figure>

you can always view and monitor the image builder process by viewing the image and use the [log_tail](https://github.com/pivotal/kpack/blob/master/docs/logs.md) utility to view the builder logs:

```bash
kubectl get images --namespace cf-system
logs --image caf42222-dc42-45ce-b11e-7f81ae511e06 --namespace cf-system
```

<figure class="full">
	<img src="/images/image_and_log.png" alt="">
	<figcaption>kubectl-image-log</figcaption>
</figure>
