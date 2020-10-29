---
layout: post
title: deploy and run DELLEMC Powerprotect Datamanager on vSphere
description: "the easiest way to get a secure container registry on Azurestack"
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

Before we start the deployment, wiwe need to check that we have
 - govc >= 0.23 insalled from [Github Releases](https://github.com/vmware/govmomi/releases/download/v0.23.0/govc_windows_amd64.exe.zip) installed in a path as govc
 - my Powershell modules for PPDM installed from [PPDM Powershell](https://www.powershellgallery.com/packages/PPDM-pwsh)using  
 ```Powershell
 install-module PPDM-pwsh
```

From a Powershell, we fisrt need to connect to our vSphere Virtual Center By using the following code,
we can securly create a connection:

```Powershell
# Set the Basic Parameter
$env:GOVC_URL="vcsa1.home.labbuildr.com"    # replace ith your vCenter
$env:GOVC_INSECURE="true"                   # allow untrusted certs
$env:GOVC_DATASTORE="vsanDatastore"         # set the default Datastore 
# read Password
$username = Read-Host -Prompt "Please Enter Virtual Center Username"
$SecurePassword = Read-Host -Prompt "Enter Password for user $username" -AsSecureString
$Credentials = New-Object System.Management.Automation.PSCredential($username, $Securepassword)
#Set Username and Password in environment
$env:GOVC_USERNAME=$($Credentials.GetNetworkCredential().username)
$env:GOVC_PASSWORD=$($Credentials.GetNetworkCredential().password)
govc ls
```
<script src="https://gist.github.com/bottkars/920fb2c16104bf0494ba9739bd383e69.js"></script>

## Testing Deployment and Parameters

<figure class="full">
	<img src="/images/connect_vc.ps1.png" alt="">
	<figcaption>connect_vc.ps1</figcaption>
</figure>

