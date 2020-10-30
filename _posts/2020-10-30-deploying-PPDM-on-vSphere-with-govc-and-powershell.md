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

<script src="https://gist.github.com/bottkars/920fb2c16104bf0494ba9739bd383e69.js"></script>

## Testing Deployment and Parameters

<figure class="full">
	<img src="/images/connect_vc.ps1.png" alt="">
	<figcaption>connect_vc.ps1</figcaption>
</figure>

