---
layout: post
title: "Create DellEMC PowerProtect Datamanager Appliance in Azure using az cli"
description: "know the basics"
modified: 2021-02-19
comments: true
published: false
tags: [aks-engine, json, bash, csi, aks, AzureStackHub]
image:
  path: /images/appliance_fresh.png
  feature: appliance_fresh.png
  credit: 
  creditlink: 
---
# basic install for test and demo or just in case
*Disclaimer: use this at your own Risk*  

## why that ?
Due to an Azure Marketplace API Change we are currently investigating, some combined Marketplace Applications in Azure no longer deploy.
This also affect´s DellEMC´s Combined Deployments for Avamar w./DataDomain as well as PowerProtect w./ DataDomain.  
This Example will deploy a Fresh PowerProtect Standalone Appliance into a Fresh Resource group for basic testing.   
This guide is to give you an understanding on how to accept marketplace terms and deploy anb appliance from a marketplace image using the cli.   
I will create a more comprehensive guide for Custom Installs soon.  

## The Problem lok like this
Uuups, an API Changed. So the blade does not give us a better answer then opening a case.   
<figure class="full">
	<img src="/images/marketplace_fail.png" alt="">
	<figcaption>Marketplace Fail</figcaption>
</figure>

so with that, we would not be able to deploy the appliance from the Marketplace.  

However, we could still use terraform, ARM Templates or ... az cli.   

So for quick and dirty, I just guide you through an az-cli process:  

## identifying the template

first of all, we need to get the current image offer for dellemc powerprotect

{% highlight shell %}
az vm image list --all --offer ppdm --publisher dellemc
{% endhighlight %}
<figure class="full">
	<img src="/images/az_image_list.png" alt="">
	<figcaption>Show images</figcaption>
</figure>

This will show you all required information to deploy the image. But before we start, we need to accept the Marketplace Image Terms for that image.  
For that, we just use *az vm image terms accept*  with the urn:    

{% highlight shell %}
az vm image terms accept --urn dellemc:ppdm_0_0_1:powerprotect-data-manager-19-7-0-7:19.7.0
{% endhighlight %}
<figure class="full">
	<img src="/images/image_term_accept.png" alt="">
	<figcaption>image term accept</figcaption>
</figure>

We also might want to look on what the image includes as image resources:  

{% highlight shell %}
az vm image show --urn dellemc:ppdm_0_0_1:powerprotect-data-manager-19-7-0-7:19.7.0
{% endhighlight %}
<figure class="full">
	<img src="/images/az_image_show.png" alt="">
	<figcaption>image show</figcaption>
</figure>


## Deploying the Image

first of all we need to create a resource group for ppdm, if not deploying to an existing one.
in my example, I am going to deploy into a resource group ppdm_from_cli in location germanywestcentral

if you do not ave already created a resource group for you deployment, do so with 
{% highlight shell %}
export RESOURCE_GROUP="ppdm_from_cli"
export LOCATION="germanywestcentral"
az group create --resource-group ${RESOURCE_GROUP} \
--location ${LOCATION}
{% endhighlight %}

<figure class="full">
	<img src="/images/az_group_create.png" alt="">
	<figcaption>az group create</figcaption>
</figure>


next, we will cerate the VM form the Marketplace image using *az vm create*.
*az vm create* will create all required resources ( vnet, NIC, NSG, pulicIp ) unless we specify a specific configuration.  
Just for the test, we will accept the standard Parameters. In Real World Scenarios, people would deploy to an existing Network that might be managed by CloudAdmin´s.  
You will find a more comprehensive Guide on that here soon.  
if you do not want a Public IP at this point, add *--public-ip-address ""*

for now, we just do:

{% highlight shell %}
export VM_NAME="ppdm1"
az vm create --resource-group ${RESOURCE_GROUP} \
 --name ${VM_NAME} \
 --image dellemc:ppdm_0_0_1:powerprotect-data-manager-19-7-0-7:19.7.0 \
 --plan-name powerprotect-data-manager-19-7-0-7 \
 --plan-product ppdm_0_0_1 \
 --plan-publisher dellemc \
 --size Standard_D8s_v3
{% endhighlight %}

Note: the required for PPDM is a Standard_D8s_v3 ! do not change that !  
<figure class="full">
	<img src="/images/az_vm_create.png" alt="">
	<figcaption>az vm create</figcaption>
</figure>

in order to access the vm via Public IP for Configuration, we need to open 443 on the NSG.
The default NSG is "${VM_NAME}NSG"

{% highlight shell %}
az network nsg rule create --name https \
--nsg-name "${VM_NAME}NSG" \
--resource-group ${RESOURCE_GROUP} \
--protocol Tcp --priority 300 \
--destination-port-range '443'
{% endhighlight %}

<figure class="full">
	<img src="/images/az_nsg_rule.png" alt="">
	<figcaption>az nsg rule</figcaption>
</figure>

Give the System a few moments to but up and configure basic things  
Meanwhile, you might want to look at you deployed resources from the Portal:  

<figure class="full">
	<img src="/images/resources_from_portal.png" alt="">
	<figcaption>resources from portal</figcaption>
</figure>

Try to connect to the appliance on https://[public_ip] after some minutes.
it should bring you to the Appliance Fresh Install page

<figure class="full">
	<img src="/images/appliance_fresh.png" alt="">
	<figcaption>appliance fresh install</figcaption>
</figure>

You can now proceed to configure the appliance. For this, follow the [PowerProtect Data Manager 19.7 Azure Deployment Guide](https://dl.dell.com/content/docu102387_PowerProtect%20Data%20Manager%2019.7%20Azure%20Deployment%20Guide.pdf?language=en_US) from our support site.  

if you want to delete you deployment just use 

{% highlight shell %}
az group delete --resource-group ${RESOURCE_GROUP}
{% endhighlight %}
