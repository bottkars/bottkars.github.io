---
layout: post
title: "Update DELL PowerProtect Datamanager Appliance using Ansible Roles and Playbooks"
description: "know the basics"
modified: 2022-07-07
comments: true
published: true
tags: [ansible, json, rest, PowerProtect]
image:
  path: /images/update_banner.png
  feature: update_banner.png
  credit: 
  creditlink: 
---
# Update Dell PowerProtect Datamanager Appliance using Ansible and REST API
*Disclaimer: use this at your own Risk*  

## why that ?
Automation is everywhere. Automation should your standards. Automation should be based on (Rest) API´s.  
As our customers broadly use Ansible to automate standard IT Tasks, i created this Example use case to manage Dell PowerProtect Datamanager´s Update lifecycle.   
My Friend Preston De Guise has written a nice Blogpost on [PowerProtect Data Manager 19.11 – What’s New, and Updating](https://nsrd.info/blog/2022/07/05/powerprotect-data-manager-19-11-whats-new-and-updating/)

We will do all of his outlined steps to update a PowerProtect Datamanager from 19.10 to 19.11 using Ansible.

The individual API Calls are Organized in Ansible roles and will be executed from corresponding Playbooks

## what we need
In general, all of the following playbooks / roles are built with the Ansible URI module ( with one exception :-) )
They have been tested from Ubuntu Linux ( running from WSL2 ), best "Ubuntu 20.04.4 LTS" for python 3.8 support

## where to get the REST API calls
we can find the REST API calls required / used in my Examples on [DELL Technologies Developer Portal](https://developer.dell.com/apis/4378/versions/19.11/reference/ppdm-public.oas2.yaml/paths/~1api~1v2~1update-packages/get)

i am happy to share my ansible roles per request for now.
so let us walk trough a typical update scenario
## 1. Uploading the Update Package
### 1.1 the *get_ppdm_token* role to authenticate with the api endpoint
All of the playbooks require to authenticate via the API endpoint and use a Bearer token for Subsequent requests.
so i use a role called get_ppdm_token to retrieve the token from the API:

{% highlight yaml %}
# role to get a Bearer Token from the PPDM API
- name: Get PPDM Token
  uri:
    method: POST
    url: "{{ ppdm_baseurl }}:{{ ppdm_port }}{{ ppdm_api_ver }}/login"
    body:
      username: "{{ ppdm_username }}"
      password: "{{ ppdm_password }}"
    body_format: json 
    status_code: 200
    validate_certs: false
  register: result  
  when: not ansible_check_mode 
- set_fact:
    access_token: "{{ result.json.access_token }}"
{% endhighlight %}


### 1.2 the *upload_update* role for uploading a package to PPDM
to upload an update package, we have to use a curl via an ansible shell call ( this being the one exception :-) ), as an upload vi URI Module fails for files > 2GB
{% highlight shell %}
# note: using curl here as uploads >=2GB still fail in ansible uri module ....
- name: Uploading Update File {{ upload_file }}, this may take a while
  shell: 'curl -ks {{ ppdm_baseurl }}:{{ ppdm_port }}{{ ppdm_api_ver }}/upgrade-packages \
      -XPOST \
      -H "content-type: multipart/form-data"  \
      -H "authorization: Bearer {{ access_token }}" \
      --form file=@{{ upload_file }}'
{% endhighlight %}


### 1.3 Playbook to upload the Update Package
The following snippet is an example of an Upload Playbook

{% highlight yaml %}
{% raw %}
# Example Playbook to configure PPDM Certificates
- name: Update PPDM
  hosts: localhost
  connection: local
  vars_files: 
    - ./vars/main.yml
  tasks:
  - name: Setting Base URL
    set_fact: 
      ppdm_baseurl: "https://{{ ppdm_fqdn | regex_replace('^https://') }}"     
  - name: Get PPDM Token for https://{{ ppdm_fqdn | regex_replace('^https://') }}
    include_role: 
      name: get_ppdm_token
    vars: 
      ppdm_password: "{{ ppdm_new_password }}"
  - debug: 
      msg: "{{ access_token }}"
      verbosity: 1
    name: do we have a token ?  
  - name: find PowerProtect update software at input mapping
    find:
      paths: "{{ lookup('env','PWD') }}/files"
      patterns: 'dellemc-ppdm-upgrade-sw-*.pkg'
    register: result
  - name: Setting Upload Filename
    set_fact:
      upgrade_file: "{{ result.files[0].path }}"
    when: result.files is defined     
  - name: parsed upload file
    debug: 
      msg: "{{ upgrade_file }}"
  - name: Uploading Update
    include_role: 
      name: upload_update
    vars: 
      upload_file: "{{ upgrade_file }}"   
{% endraw %}      
{% endhighlight %}

The Playbook will run like this:

<figure class="full">
	<img src="/images/upload_ppdm_upgrade.png" alt="">
	<figcaption>Playbook Upload Update</figcaption>
</figure>

## 2. Pre Checking the Update

The Update Pre Check will inform you about Required Actions need to be taken in order to make the update succeed.
It will also show us Warnings for issues we might correct prior the Update.

### 2.1 the *get_ppdm_update_package* role for getting the Update ID
PPDM has an API endpoint to get all Updates in the System. This could be active or historical Updates.
The Response Contains JSON Documents with specific information and state about the Update

{% highlight yaml %}
{% raw %}
- name: Define uri with package ID
  set_fact: 
    uri: "{{ ppdm_baseurl }}:{{ ppdm_port }}{{ ppdm_api_ver }}/upgrade-packages/{{ id }}"
  when: id  | default('', true) | trim != ''
- name: Define uri without package id
  set_fact: 
    uri: "{{ ppdm_baseurl }}:{{ ppdm_port }}{{ ppdm_api_ver }}/upgrade-packages"
  when: id  | default('', false) | trim == ''
- name: Rewrite uri with filter
  set_fact: 
    uri: "{{ uri }}?filter={{ filter | urlencode }}"  
  when: filter  | default('', true) | trim != ''
- name: Getting Update Package State
  uri:
    method: GET
    url: "{{ uri }}"
    body_format: json
    headers:
      accept: application/json
      authorization: "Bearer {{ access_token }}"    
    status_code: 200,202,403
    validate_certs: false
  register: result  
  when: not ansible_check_mode 
- set_fact:
    update_package: "{{ result.json.content }}"
  when: id  | default('', false) | trim == ''
- set_fact:
    update_package: "{{ result.json }}"
  when: id  | default('', true) | trim != ''      
- debug:
    msg: "{{ update_package }}"
    verbosity: 1
{% endraw %}      
{% endhighlight %}

This task allows us to get info a specific update by using the update ID, or filter an update by type ( e.g. ACTIVE or HISTORY )

### 2.2 example Playbook to get an ACTIVE Update Content 

{% highlight yaml %}
{% raw %}
# Example Playbook to get an ACTIVE Package
#
# Note: Api is still called Upgrade, but Upgrade is for Hardware only !
# SO tasks will be named update but execute Upgrade
#
- name: get PPDM Update Package
  hosts: localhost
  connection: local
  vars_files: 
    - ./vars/main.yml
  tasks:
  - name: Setting Base URL
    set_fact: 
      ppdm_baseurl: "https://{{ ppdm_fqdn | regex_replace('^https://') }}"     
  - name: Get PPDM Token for https://{{ ppdm_fqdn | regex_replace('^https://') }}
    include_role: 
      name: get_ppdm_token
    vars: 
      ppdm_password: "{{ ppdm_new_password }}"
  - debug: 
      msg: "{{ access_token }}"
      verbosity: 1
    name: do we have a token ?  
  - name: get Update Package
    include_role: 
      name: get_ppdm_update_package
    vars:
      filter: 'category eq "ACTIVE"' 
  - debug: 
      msg: "{{ update_package }}"

{% endraw %}      
{% endhighlight %}

This example shows the JSON response for the Update Package with Essential information for example Versions,Required Passwords/Reboot, Updated EULA´s, ID etc:
<figure class="full">
	<img src="/images/active_upgrade_package.png" alt="">
	<figcaption>Example Upgrade Package</figcaption>
</figure>

### 2.3 the *precheck_ppdm_update_package* role
After reviewing the Information, we will trigger the Pre Check using the precheck endpoint.
This will transform the Package state to Processing.
The below Precheck role will wait for the state AVAILABLE and the Validation Results: 
{% highlight yaml %}
{% raw %}
# Example Playbook to Precheck PPDM Update
#
# Note: Api is still called Upgrade, but Upgrade is for Hardware only !
# SO tasks will be named update but execute Upgrade
#
- name: precheck PPDM Update Package
  uri:
    method: POST
    url: "{{ ppdm_baseurl }}:{{ ppdm_port }}{{ ppdm_api_ver }}/upgrade-packages/{{ id }}/precheck"
    body_format: json
    headers:
      accept: application/json
      authorization: "Bearer {{ access_token }}"    
    status_code: 202,204,401,403,404,409,500
    validate_certs: false
  register: result  
  retries: 10
  delay: 30
  until: result.json.state is defined and result.json.state == "PROCESSING"
  when: not ansible_check_mode  
- set_fact:
    update_precheck: "{{ result.json }}"
- debug:
    msg: "{{ update_precheck }}"
    verbosity: 1
- name: Wait  Update Package State
  uri:
    method: GET
    url: "{{ ppdm_baseurl }}:{{ ppdm_port }}{{ ppdm_api_ver }}/upgrade-packages/{{ id }}"
    body_format: json
    headers:
      accept: application/json
      authorization: "Bearer {{ access_token }}"    
    status_code: 200,202,403
    validate_certs: false
  register: result 
  retries: 10 
  delay:  10
  when: not ansible_check_mode 
  until: result.json.validationDetails is defined and result.json.state == "AVAILABLE"
- set_fact:
    validation_details: "{{ result.json.validationDetails }}"   
- debug:
    msg: "{{ validation_details }}"
    verbosity: 1    
{% endraw %}      
{% endhighlight %}



### 2.4 Running the Precheck Playbook
The Corresponding Playbook for above task will may look like this:
{% highlight yaml %}
{% raw %}
# Example Playbook to Precheck PPDM Update
#
# Note: Api is still called Upgrade, but Upgrade is for Hardware only !
# SO tasks will be named update but execute Upgrade
#
- name: Upgrade PPDM
  hosts: localhost
  connection: local
  vars_files: 
    - ./vars/main.yml
  tasks:
  - name: Setting Base URL
    set_fact: 
      ppdm_baseurl: "https://{{ ppdm_fqdn | regex_replace('^https://') }}"     
  - name: Get PPDM Token for https://{{ ppdm_fqdn | regex_replace('^https://') }}
    include_role: 
      name: get_ppdm_token
    vars: 
      ppdm_password: "{{ ppdm_new_password }}"
  - debug: 
      msg: "{{ access_token }}"
      verbosity: 1
    name: do we have a token ?  
  - name: get Update Package
    include_role: 
      name: get_ppdm_update_package
    vars:
      filter: 'category eq "ACTIVE"' 

  - name: delete_ppdm_upgrade
    when: update_package[0] is defined and update_package[0].upgradeError is defined
    include_role: 
      name: delete_ppdm_update_package
    vars:
      id: "{{ update_package[0].id }}"
  - name: precheck_ppdm_upgrade
    when: update_package[0] is defined and update_package[0].upgradeError is not defined
    include_role: 
      name: precheck_ppdm_update_package
    vars:
      id: "{{ update_package[0].id }}"
  - debug:
      msg: "{{ validation_details }}"
      verbosity: 0          
{% endraw %}      
{% endhighlight %}
The Playbook  will output the validation Details:
<figure class="full">
	<img src="/images/validation_details.png" alt="">
	<figcaption>Validation Details</figcaption>
</figure>


## 3. Executing the Upgrade
Once the Update Validation has passed, we can execute the Update
### 3.1 the *install_ppdm_update_package* role
The below example task shows the upgrade-packages endpoint to be called
A JSON Body needs to be provided with additional information.
We wil create the Body from the Calling Playbook 

{% highlight yaml %}
{% raw %}
- name: install PPDM Update Package
  uri:
    method: PUT
    url: "{{ ppdm_baseurl }}:{{ ppdm_port }}{{ ppdm_api_ver }}/upgrade-packages/{{ id }}?forceUpgrade=true"
    body_format: json
    body: "{{ body }}"
    headers:
      accept: application/json
      authorization: "Bearer {{ access_token }}"    
    status_code: 202,204,401,403,404,409,500
    validate_certs: false
  register: result  
  when: not ansible_check_mode  
- set_fact:
    update: "{{ result.json }}"
- debug:
    msg: "{{ update }}"
    verbosity: 0
{% endraw %}      
{% endhighlight %}

### 3.2 the *check_update_done* role
Once the Update is started, PPDM sysmgr and other components will shut down.
PPDM has a specific API Port to Monitor the Update( also from the UI, Port 14443)
The check_update_done role will wait until the Update reaches 100%


{% highlight yaml %}
{% raw %}
- name: Wait for Appliance Update Done, this may take a few Minutes
  uri:
    method: GET
    url: "{{ ppdm_baseurl }}:14443/upgrade/status"
    status_code: 200 
    validate_certs: false
  register: result  
  when: not ansible_check_mode 
  retries: 100
  delay: 30
  until: ( result.json is defined and result.json[0].percentageCompleted == "100" ) or ( result.json is defined and result.json[0].upgradeStatus =="FAILED" )
- debug: 
    msg: "{{ result.json }}"
    verbosity: 0
  when: result.json is defined  
{% endraw %}      
{% endhighlight %}
### 3.3 Playbook to install the Update
to install the update, we will modify the update-package JSON document to change the state field to INSTALLED.
Also, we will approve updated EULA Settings and trust the Package Certificate.
The changes will be merged from the Playbook. 

{% highlight yaml %}
{% raw %}
state: INSTALLED
certificateTrustedByUser: true
eula: 
  productEulaAccepted: true # honor EULA Changes
  telemetryEulaAccepted: true  # honor EULA Changes
{% endraw %}      
{% endhighlight %}

The corresponding Playbook looks like
{% highlight yaml %}
{% raw %}
# Example Playbook to Install PPDM Update
#
# Note: Api is still called Upgrade, but Upgrade is for Hardware only !
# SO tasks will be named update but execute Upgrade
#
- name: Upgrade PPDM
  hosts: localhost
  connection: local
  vars_files: 
    - ./vars/main.yml
  tasks:
  - name: Setting Base URL
    set_fact: 
      ppdm_baseurl: "https://{{ ppdm_fqdn | regex_replace('^https://') }}"     
  - name: Get PPDM Token for https://{{ ppdm_fqdn | regex_replace('^https://') }}
    include_role: 
      name: get_ppdm_token
    vars: 
      ppdm_password: "{{ ppdm_new_password }}"
  - debug: 
      msg: "{{ access_token }}"
      verbosity: 1
    name: do we have a token ?  
  - name: get Update Package
    include_role: 
      name: get_ppdm_update_package
    vars:
      filter: 'category eq "ACTIVE"' 

  - name: get_ppdm_upgrade_by ID
    when: update_package[0] is defined and update_package[0].upgradeError is not defined
    include_role: 
      name: get_ppdm_update_package
    vars:
      id: "{{ update_package[0].id }}"

  - name: merge update body
    vars:
      my_new_config:
        state: INSTALLED
        certificateTrustedByUser: true
        eula: 
          productEulaAccepted: true # honor EULA Changes
          telemetryEulaAccepted: true  # honor EULA Changes
    set_fact:
      update_package: "{{ update_package | default([]) | combine(my_new_config, recursive=True) }}"
  - debug: 
      msg: "{{ update_package }}"
  - name: install_ppdm_upgrade
    when: update_package is defined and update_package.state == "INSTALLED"
    include_role: 
      name: install_ppdm_update_package
    vars:
      id: "{{ update_package.id }}"
      body: "{{ update_package }}"
  - name: Validate Update Started
    when: update_package is defined and update_package.upgradeError is not defined
    include_role: 
      name: get_ppdm_update_package
    vars:
      id: "{{ update_package.id }}"   
  - name: Check Update State  https://{{ ppdm_fqdn | regex_replace('^https://') }}
    include_role:
      name: check_update_done 
{% endraw %}      
{% endhighlight %}
We can follow the Upgrade Process from the Ansible Playbook:
<figure class="full">
	<img src="/images/install_update.png" alt="">
	<figcaption>Update Playbook</figcaption>
</figure>

And also use the Webbrowser at "https://<ppdm_fqdn>:14443" to see the Update Status:
<figure class="full">
	<img src="/images/update_status.png" alt="">
	<figcaption>Update Status UI</figcaption>
</figure>
