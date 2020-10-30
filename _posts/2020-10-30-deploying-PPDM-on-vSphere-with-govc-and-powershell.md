---
layout: post
title: deploy and run DELLEMC Powerprotect Datamanager on vSphere
description: "the fastest way to get tarted with PPDM on vSphere"
modified: 2020-10-30
comments: true
published: true
tags: [Powerprotect, vSphere, Powershell, PPDM, PPDM-pwsh]
image:
  path: /images/connect_vc.ps1.png
  feature: docker_push.png
  credit: 
  creditlink: 
---

# Deploying PowerProtect Datamanger (PPDM) to vSpger using govc and Powershell 

this is an explanation of my Deployment of PPDM to vShere using vmware govc and my [PPDM Powershell](https://www.powershellgallery.com/packages/PPDM-pwsh/)  Module
## Requirements
Before we start the deployment, wiwe need to check that we have
 - govc >= 0.23 insalled from [Github Releases](https://github.com/vmware/govmomi/releases/download/v0.23.0/govc_windows_amd64.exe.zip) installed in a path as govc
 - my Powershell modules for PPDM installed from [PPDM Powershell](https://www.powershellgallery.com/packages/PPDM-pwsh)using  
 {% highlight scss %}
 install-module PPDM-pwsh
{% endhighlight %}
## Step 1: Connecting to vSphere using govc
From a Powershell, we first need to connect to our vSphere Virtual Center By using the following code,
we can securly create a connection:

<script src="https://gist.github.com/bottkars/920fb2c16104bf0494ba9739bd383e69.js"></script>



<figure class="full">
	<img src="/images/connect_vc.ps1.png" alt="">
	<figcaption>connect_vc.ps1</figcaption>
</figure>

## Step 2:deploying Powerprotect Datamanager ova using govc from Powershell
download the latest Powerprotect DataManager from [DELLEMC Support](https://dl.dell.com/downloads/DL100787_PowerProtect-Data-Manager-19.6-Install-OVA.ova) *login required*

first of all, we set our govc environment to have the Following Variables
( complete codesnipped of step 2 below )
{% highlight scss %}
$ovapath="$HOME/Downloads/dellemc-ppdm-sw-19.6.0-3.ova" # the Path to your OVA File
$env:GOVC_FOLDER='/home_dc/vm/labbuildr_vms'            # the vm Folder in your vCenter wher the Machine can be found
$env:GOVC_VM='ppdm_demo'                                # the vm Name
$env:GOVC_DATASTORE='mgmtvms'                           # The Name of the Datastore
$env:GOVC_HOST='esxi-mgmt.home.labbuildr.com'           # The taget ESXi Host for Deployment
{% endhighlight %}
then we need to import the Virtual Appliance Specification from the ova using *govc import.spec*
the command would look like

{% highlight scss %}
$SPEC=govc import.spec $ovapath| ConvertFrom-Json
{% endhighlight %}

Once we have the Configuration Data, we will change the vami keys in the "Property Mappings" to our desired Values

{% highlight scss %}
# edit you ip address
$SPEC.PropertyMapping[0].Key='vami.ip0.brs'
$SPEC.PropertyMapping[0].Value='100.250.1.123'
# Default Gateway
$SPEC.PropertyMapping[1].Key='vami.gateway.brs'
$SPEC.PropertyMapping[1].Value = "100.250.1.1" 
# Subnet Mask               
$SPEC.PropertyMapping[2].Key = "vami.netmask0.brs"
$SPEC.PropertyMapping[2].Value = "255.255.255.0" 
# DNS Servers
$SPEC.PropertyMapping[3].Key = "vami.DNS.brs"
$SPEC.PropertyMapping[3].Value = "192.168.1.44" 
# you FQDN, make sure it is resolvable from above DNS
$SPEC.PropertyMapping[4].Key = "vami.fqdn.brs"
$SPEC.PropertyMapping[4].Value = "ppdmdemo.home.labbuidr.com"   
{% endhighlight %}
Now we need to import the OVA with the settings we just created:

{% highlight scss %}
$SPEC | ConvertTo-Json | Set-Content -Path spec.json
govc import.ova -name $env:GOVC_VM -options="./spec.json" $ovapath
{% endhighlight %}
And change to the Correct "VM Network" for  ethernet-0
{% highlight scss %}
govc vm.network.change -net="VM Network" ethernet-0
{% endhighlight %}

Now, we can Power on the vm with 

{% highlight scss %}
govc vm.power -on $env:GOVC_VM
{% endhighlight %}





{% highlight scss %}

{% endhighlight %}

### The complete script can be found here:

<script src="https://gist.github.com/bottkars/f05d8357232778a24da45a46eb382a3d.js"></script>


<figure class="full">
	<img src="/images/connect_vc.ps1.png" alt="">
	<figcaption>connect_vc.ps1</figcaption>
</figure>

## Step 3