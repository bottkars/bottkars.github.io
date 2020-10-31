---
layout: post
title: Deploy Powerprotect Datamanager on vSphere using govc and Powershell
description: "the fastest way to get tarted with PPDM on vSphere"
modified: 2020-10-30
comments: true
published: true
tags: [Powerprotect, vSphere, Powershell, PPDM, PPDM-pwsh]
image:
  path: /images/config-success.png
  feature: docker_push.png
  credit: 
  creditlink: 
---
DRAFT
# Deploying PowerProtect Datamanager (PPDM) to vSphere using govc and Powershell 
when it comes to deploy Powerprotect Datamanger, we have a variety of options, for example
- Terraform
- Ansible
- OVA deployment from vCenter UI
- Saltstack
- bash / Concourse
just to name few.
In this Post I focus on a Powershell Deployment leveraging mware govc and my [PPDM Powershell](https://www.powershellgallery.com/packages/PPDM-pwsh/) Module

## Requirements
Before we start the deployment, we need to check that we have
 - govc >= 0.23 insalled from [Github Releases](https://github.com/vmware/govmomi/releases/download/v0.23.0/govc_windows_amd64.exe.zip) installed in a path as govc
 - my Powershell modules ( minimum : 0.2.8 ) for PPDM installed from [PPDM Powershell](https://www.powershellgallery.com/packages/PPDM-pwsh)using  
 {% highlight scss %}
 install-module PPDM-pwsh -MinimumVersion 0.2.8
{% endhighlight %}
## Step 1: Connecting to vSphere using govc
From a Powershell, we first need to connect to our vSphere Virtual Center By using the following code,
we can securely create a connection:
{% highlight scss %}
# Set the Basic Parameter
$env:GOVC_URL="vcsa1.home.labbuildr.com"    # replace ith your vCenter
$env:GOVC_INSECURE="true"                   # allow untrusted certs
$env:GOVC_DATASTORE="vsanDatastore"         # set the default Datastore 
# read Password
$username = Read-Host -Prompt "Please Enter Virtual Center Username default (Administrator@vsphere.local)"
If(-not($username)){$username = "Administrator@vsphere.local"}
$SecurePassword = Read-Host -Prompt "Enter Password for user $username" -AsSecureString
$Credentials = New-Object System.Management.Automation.PSCredential($username, $Securepassword)
#Set Username and Password in environment
$env:GOVC_USERNAME=$($Credentials.GetNetworkCredential().username)
$env:GOVC_PASSWORD=$($Credentials.GetNetworkCredential().password)
govc about
{% endhighlight %}


<figure class="full">
	<img src="/images/connect_vc.ps1.png" alt="">
	<figcaption>connect_vc.ps1</figcaption>
</figure>

## Step 2:deploying Powerprotect Datamanager ova using govc from Powershell
- Requirement:
download the latest Powerprotect DataManager from [DELLEMC Support](https://dl.dell.com/downloads/DL100787_PowerProtect-Data-Manager-19.6-Install-OVA.ova) ( *login required* )

first of all, we set our govc environment to have the Following Variables
( complete code snippet of step 2 below )
{% highlight scss %}
$ovapath="$HOME/Downloads/dellemc-ppdm-sw-19.6.0-3.ova" # the Path to your OVA File
$env:GOVC_FOLDER='/home_dc/vm/labbuildr_vms'            # the vm Folder in your vCenter where the Machine can be found
$env:GOVC_VM='ppdm_demo'                                # the vm Name
$env:GOVC_DATASTORE='mgmtvms'                           # The Name of the Datastore
$env:GOVC_HOST='esxi-mgmt.home.labbuildr.com'           # The target ESXi Host for Deployment
{% endhighlight %}
then we need to import the Virtual Appliance Specification from the ova using *govc import.spec*
the command would look like

{% highlight scss %}
$SPEC=govc import.spec $ovapath| ConvertFrom-Json
{% endhighlight %}

Once we have the Configuration Data, we will change the vami keys in the "Property Mappings" to our desired Values

{% highlight scss %}
# edit your ip address
$SPEC.PropertyMapping[0].Key='vami.ip0.brs'
$SPEC.PropertyMapping[0].Value='100.250.1.123' # < your IP here
# Default Gateway
$SPEC.PropertyMapping[1].Key='vami.gateway.brs'
$SPEC.PropertyMapping[1].Value = "100.250.1.1" # < your Gateway here
# Subnet Mask               
$SPEC.PropertyMapping[2].Key = "vami.netmask0.brs"
$SPEC.PropertyMapping[2].Value = "255.255.255.0" # < your Netmask here
# DNS Servers
$SPEC.PropertyMapping[3].Key = "vami.DNS.brs"
$SPEC.PropertyMapping[3].Value = "192.168.1.44" # < your DNS Server here
# you FQDN, make sure it is resolvable from above DNS
$SPEC.PropertyMapping[4].Key = "vami.fqdn.brs"
$SPEC.PropertyMapping[4].Value = "ppdmdemo.home.labbuidr.com" # < your Gateway here   
{% endhighlight %}
Now we need to import the OVA using *govc import.ova* with the settings we just created:

{% highlight scss %}
$SPEC | ConvertTo-Json | Set-Content -Path spec.json
govc import.ova -name $env:GOVC_VM -options="./spec.json" $ovapath
{% endhighlight %}
And change to the Correct "VM Network" for  ethernet-0
{% highlight scss %}
govc vm.network.change -net="VM Network" ethernet-0
{% endhighlight %}

Now, we can Power on the vm using *govc vm.power* ...

{% highlight scss %}
govc vm.power -on $env:GOVC_VM
{% endhighlight %}

... and wait for the Powerprotect Datamanager Services to be up and running.

In an Automated Scenario, one could query the http://fqdn.of.ppdm:443/#/fresh until receiving a 200

## Step 3: Configure PPDM using PPDM-pwsh

if not already node, load the Modules by 

{% highlight scss %}
import-module PPDM-pwsh
{% endhighlight %}

The first step is to connect to the PPDM API.
You will be asked for for the username ad *admin* and Password of *admin* 
We will retrieve a Bearer Token from the API, that will be used automatically for Subsequent requests in the Current Powershell Session. 
PPDM-pwsh also will figure out your Powershell Version and therefore use different Methods to use non trusted certificates. ( -trustCert)

{% highlight scss %}
$API=Connect-PPDMapiEndpoint -PPDM_API_URI https://ppdm-demo.home.labbuildr.com -user -trustCert
{% endhighlight %}


Once connected, we need to Accept the EULA for PPDM by using
{% highlight scss %}
Approve-PPDMEula
{% endhighlight %}

The next step is to configure PPDM. For that, we need to specify Timezone, NTP Server and the new Password(s)
to get a list of timezones, run
{% highlight scss %}
Get-PPDMTimezones
{% endhighlight %}

In the below example, we use Europe/Berlin :

<figure class="full">
	<img src="/images/connect_timezome.png" alt="">
	<figcaption>EULA and Timezone</figcaption>
</figure>


{% highlight scss %}
Set-PPDMconfigurations -NTPservers 139.162.149.127 -Timezone "Europe/Berlin" -admin_Password 'Password123!'
{% endhighlight %}
<figure class="full">
	<img src="/images/set-ppdmconfigurations.png" alt="">
	<figcaption>set-ppdmconfigurations</figcaption>
</figure>
It will take up to 10 Minutes for PPDM to finish. Monitor with 



{% highlight scss %}
 Get-PPDMconfigurations | Get-PPDMconfigstatus
{% endhighlight %}

<figure class="full">
	<img src="/images/config-success.png" alt="">
	<figcaption>config success</figcaption>
</figure>


You can now visit the PPDM Homepage from your Webbrowser to configure DataDomain, add a vCenter, add Kubernetes Clusters and more.
In my next Post we will do so as well from Powershell. stay Tuned


<figure class="full">
	<img src="/images/ppdm-wizard.png" alt="">
	<figcaption>config success</figcaption>
</figure>

### Scriptlistings:
<script src="https://gist.github.com/bottkars/920fb2c16104bf0494ba9739bd383e69.js"></script>
<script src="https://gist.github.com/bottkars/f05d8357232778a24da45a46eb382a3d.js"></script>


