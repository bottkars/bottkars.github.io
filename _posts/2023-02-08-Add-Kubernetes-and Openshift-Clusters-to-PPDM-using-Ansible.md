---
layout: post
title: "Add K8S Clusters to PowerProtect Datamanager using Rest and Ansible"
description: "know the basics"
modified: 2023-02-19
comments: true
published: false
tags: [OpenShift, yaml, ansible, PowerProtect Datamanager, aks, Kubernetes]
image:
  path: /images/appliance_fresh.png
  feature: appliance_fresh.png
  credit: 
  creditlink: 
---
# This is based on my ansible-ppdm roles
*for complete roles access please dm me @azurestack_guy*  

## why that ?
As Kubernetes Clusters are generated from Code these days ( e.g. OpenShift-Installer, aks-engine, tanzu etc.) we need to automate the onboarding of Clusters into PowerProtect DataManager.

As Ansible is the d facto Standard for Managing Openshift CLusters, i will use my Ansible Roles for PPDM as well as the ansible.core.k8s modules to do the required steps

## the process
The Process is pretty straight forward in will reflect the Steps taken for a normal Kubernetes Onboarding into PPDM

those will include:
	- Adding RBAC Controls and Discovery Account for PPDM to Kubernetes
	- Importing and Accepting the CLuster Certificate to PPDM
	- Adding the Kubernetes CLuster and set Optional Parameters ( e.g. Private Registry location)
## the Playbook

###


<figure class="full">
	<img src="/images/appliance_fresh.png" alt="">
	<figcaption>appliance fresh install</figcaption>
</figure>

You can now proceed to configure the appliance. For this, follow the [PowerProtect Data Manager 19.7 Azure Deployment Guide](https://dl.dell.com/content/docu102387_PowerProtect%20Data%20Manager%2019.7%20Azure%20Deployment%20Guide.pdf?language=en_US) from our support site.  

if you want to delete you deployment just use 

{% highlight shell %}
az group delete --resource-group ${RESOURCE_GROUP}
{% endhighlight %}
