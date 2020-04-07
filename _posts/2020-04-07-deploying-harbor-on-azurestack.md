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

in the following examples, i deploy 2 Registry, 1 called devregistry with self-signed Certificates, and one called registry, to become my Production Registry using let´s encrypt Certificates

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
*note: i am using an inline variable check with :? do validate the variables are set. this is one of my best practices to not pass empty values to Parameters that are not validated / are allowed to be empty.*
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
### start deployment for selfsigned registry


```bash
az deployment group create --resource-group  ${DNS_LABEL_PREFIX:?variable is empty} \
    --template-uri "https://raw.githubusercontent.com/bottkars/201-azurestack-harbor-registry/master/azuredeploy.json" \
    --parameters \
    sshKeyData="$(cat ~/.ssh/id_rsa.pub)" \
    HostDNSLabelPrefix=${DNS_LABEL_PREFIX:?variable is empty}
```
<figure class="full">
	<img src="/images/create_devregistry_deployment.png" alt="">
	<figcaption>create_devregistry_deployment</figcaption>
</figure>  
### start the deployment for registry using your own CA Certs:
```bash
az deployment group create --resource-group ${DNS_LABEL_PREFIX:?variable is empty}\
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
	<img src="/images/create_registry_deployment.png" alt="">
	<figcaption>create_registry_deployment</figcaption>
</figure> 



### validation / monitoring the installation

You can monitor the deployment in the Azurestack User Portal. The Resource group will be the name of the *DNS_LABEL_PREFIX*

<figure class="full">
	<img src="/images/harbor_rg.png" alt="">
	<figcaption>harbor_rg</figcaption>
</figure>

once the Public IP is online, you can also ssh into the Harbor host to monitor the Custom Script execution:
```bash
ssh ubuntu@${DNS_LABEL_PREFIX:? variable empty}.local.cloudapp.azurestack.external
```


