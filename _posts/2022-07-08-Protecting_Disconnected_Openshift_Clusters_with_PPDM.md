---
layout: post
title: "Configure disconnected OpenShift Cluster´s for PowerProtect Datamanager 19.11"
description: "Tipps and Troublesooting"
modified: 2022-07-08
comments: true
published: true
tags: [OPenShift, json, rest, PowerProtect]
image:
  path: /images/operator_ppdm.png
  feature: operator_ppdm.png
  credit: 
  creditlink: 
---
### Installation and troubleshooting Tips for PPDM 19.11 in disconnected OpenShift 4.10 Environments
*Disclaimer: use this at your own Risk*  
<figure class="full">
	<img src="/images/parental_advisory.png" alt="">
	<figcaption>Warning, Contains JSON and YAML !!!</figcaption>
</figure>
## why that ?
I am happily running some disconnected OpenShift Clusters on VMware and AzureStack Hub.
This Post is about how to Protect a Greenfield Deployment of OpenShift 4.10 with PPDM 19.11, running on vSphere.

OpenShift Container Platform 4.10 installs the vSphere CSI Driver Operator and the vSphere CSI driver by default in the openshift-cluster-csi-drivers namespace.  
This setup is different from a user Deployed CSI driver or CSI Drivers as a Process like in TKGi. 

In this Post, i will point out some Changes required to make OpenShift 4.10 fresh installs work with PPDM 19.11

## 1. what we need
In order to support Disconnected OpenShift Clusters, we need to prepare our environment to Host certain Images and Operators on a local Registry.

### 1.1 Mirroring the Operator Catalog for the OADP Provider
PPDM utilized the RedHat OADP  (OpenShift APIs for Data Protection) Operator vor Backup and Restore. Therefore, we need to Replicate 
*redhat-operators* from the index image *registry.redhat.io/redhat/redhat-operator-index:v4.10*
This is well documented on [Using Operator Lifecycle Manager on restricted networks](https://docs.openshift.com/container-platform/4.10/operators/admin/olm-restricted-networks.html), specifically if you want to filter the Catalog to a specific list of Packages.

If you want to mirror the entire Catalog, you can utilize *oc adm catalog mirror*

Once the Replication is done, make sure apply the ImageContentSourcePolicy and the new CatalogSource itself.

### 1.2 Mirroring the DELL PowerProtect Datamanager and vSphere/Velero Components
The Following Images need to be Replicated to your local Registry: 

Image  | Repository |  Tag
---|---|---
dellemc/powerprotect-k8s-controller  |  https://hub.docker.com/r/dellemc/powerprotect-k8s-controller/tags  |  19.10.0-20
dellemc/powerprotect-cproxy                   |  https://hub.docker.com/r/dellemc/powerprotect-cproxy/tags  |  19.10.0-20
dellemc/powerprotect-velero-dd                |  https://hub.docker.com/r/dellemc/powerprotect-velero-dd/tags |  19.10.0-20
velero/velero                                 |  https://hub.docker.com/r/velero/velero/tags |  v1.7.1
vsphereveleroplugin/velero-plugin-for-vsphere |  https://hub.docker.com/r/vsphereveleroplugin/velero-plugin-for-vsphere/tags |  v1.3.1
vsphereveleroplugin/backup-driver             |  https://hub.docker.com/r/vsphereveleroplugin/backup-driver/tags             |  v1.3.1


## 2. Setup
### 2.1 Configure PPDM to point to the local Registry
On the PPDM Server, point the CNDM (Cloud Native Datamover )Service to the Registry that is hosting your PPDM and Velero Images by editing
*/usr/local/brs/lib/cndm/config/application.properties* 
Put the FQDN for k8s.docker.registry to reflect your local registry:  
{% highlight yaml %}
k8s.docker.registry=harbor.pks.home.labbuildr.com
{% endhighlight %}
This will instruct cndm to point to your registry when creating configmaps for PPDM
You need to restart the Service in order for changes to affect.
{% highlight shell %}
cndm restart
{% endhighlight %}

### 2.2 Configure a Storage Class with *volumeBindingMode: Immediate*
The vSphere CSI Operator Driver storage class uses vSphere’s storage policy. OpenShift Container Platform automatically creates a storage policy that targets datastore configured in cloud configuration. However, this uses *volumeBindingMode: WaitForFirstConsumer*. In order to restore PVC´s to a new Namespace, we need a StorageClass with *volumeBindingMode: Immediate*
Make sure the StoragePolicyName reflects ure existing Policy ( from Default Storage Class )
{% highlight bash %}
{% raw %}
oc apply -f - <<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: thin-csi-immediate
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: csi.vsphere.vmware.com
parameters:
  StoragePolicyName: openshift-storage-policy-ocs1-nv6xw # <lookup you policy>
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
EOF{% endraw %}      
{% endhighlight %}

### 2.3 Deploy the RBAC Templates from PPDM
RBAC Templates containing PPDM  CRD´s, RoleBindings and ClusterRoleBindings,Roles and ConfigMaps and ServiceAccount Definitions can 
found on PPDM Server in */usr/local/brs/lib/cndm/misc/rbac.tar.gz*.  
Always use the Latest version that comes with your PPDM version !  
Once you copied and extracted the Templates, apply the to your cluster:

{% highlight bash %}
oc apply -f ../ocs_vsphere/ppdm-rbac/rbac/ppdm-discovery.yaml
oc apply -f ../ocs_vsphere/ppdm-rbac/rbac/ppdm-controller-rbac.yaml
{% endhighlight %}

### 2.4 Retrieve the PPDM Discovery Service Account Token
Applying the above Templates has also created the Powerprotect namespace and Discovery Token.
Retrieve the token with
{% highlight bash %}
{% raw %}
export PPDM_K8S_TOKEN=$(oc get secret \
"$(oc -n powerprotect get secret \
 | grep ppdm-discovery-serviceaccount-token \
 | head -n1 | awk '{print $1}')" \
-n powerprotect --template={{.data.token}} | base64 -d{% endraw %}      
{% endhighlight %}

### 2.5 Add the OpenShift Cluster to PPDM
If you not have already enabled the Kubernetes Asset Source in PPDM, do so now.
To do so, go to *Infrastructure > AssetSources > +*
In the New Asset Source Tab, Click on *Enable Source* for the Kubernetes Asset Source
<figure class="full">
	<img src="/images/enable_asset_source_k8s.png" alt="">
	<figcaption>Enable the OCS Cluster</figcaption>
</figure>
Select the Kubernetes tab and click Add. Provide your Cluster information and click on Add credentials.
Provide a Name for the Discovery Service Account and Provide the Token extracted in the Previous Step
<figure class="full">
	<img src="/images/add_k8s_creds.png" alt="">
	<figcaption>Adding Credetials</figcaption>
</figure>

Click on Verify to Validate the Certificate of the OpenShift Cluster
hint: if you use unknown root ca, follow the user guide to add the ca to ppdm using the ppdmtool
<figure class="full">
	<img src="/images/add_ocs.png" alt="">
	<figcaption>Adding the OCS Cluster</figcaption>
</figure>

The  the powerprotect controller and Configmap should now being deployed to powerprotect namespace

To view the configmap, execute

{% highlight bash %}
oc get configmap  ppdm-controller-config -n powerprotect -o=yaml
{% endhighlight %}

to view the logs, execute: 
{% highlight bash %}
oc logs "$(oc -n powerprotect get pods \
| grep powerprotect-controller \
| awk '{print $1}')" -n powerprotect -f
{% endhighlight %}


if the controller is waiting for velero-ppdm ip,
<figure class="full">
	<img src="/images/wait_ip.png" alt="">
	<figcaption>Waiting for velero-ppdm</figcaption>
</figure>

then probably your oadp-operator subscription is pointing to the wrong operator catalog source.
View your subscription with
{% highlight bash %}
oc describe subscription redhat-oadp-operator -n velero-ppdm
{% endhighlight bash %}

{% highlight yaml %}
  Conditions:
    Last Transition Time:  2022-04-22T08:54:35Z
    Message:               targeted catalogsource openshift-marketplace/community-operators missing
    Reason:                UnhealthyCatalogSourceFound
    Status:                True
    Type:                  CatalogSourcesUnhealthy
    Message:               constraints not satisfiable: no operators found from catalog community-operators in namespace openshift-marketplace referenced by subscription oadp-operator, subscription oadp-operator exists
    Reason:                ConstraintsNotSatisfiable
    Status:                True
    Type:                  ResolutionFailed
  Last Updated:            2022-04-22T08:54:35Z
Events:                    <none>
{% endhighlight  %}


In this case, we need to patch the Subscription to reflect the Catalog Name we created for our custom Operator Catalog ( in this example, *redhat-operator-index*):
{% highlight bash %}
oc patch subscription redhat-oadp-operator -n velero-ppdm --type=merge -p '{"spec":{"source": "redhat-operator-index"}}'
{% endhighlight bash %}
This should deploy the RedHat OADP Operator in the velero-ppdm namespace.
You can view the Operator Status from your OpenShift Console
<figure class="full">
	<img src="/images/rh_oadp.png" alt="">
	<figcaption>Waiting for OADP Install</figcaption>
</figure>

Now it is time to watch the Pods Created in the velero-ppdm Namespace:

{% highlight bash %}
{% raw %}
oc get pods -n velero-ppdm
{% endraw %}      
{% endhighlight %}
<figure class="full">
	<img src="/images/velero-ppdm-ns.png" alt="">
	<figcaption>Waiting for Pods</figcaption>
</figure>

The Backup driver fails to deploy, so we need to have a look at the logs:
{% highlight bash %}
oc logs "$(oc -n velero-ppdm get pods | grep backup-driver | awk '{print $1}')" -n velero-ppdm -f    
{% endhighlight %}
<figure class="full">
	<img src="/images/backup_driver.png" alt="">
	<figcaption>Backup Driver Logs</figcaption>
</figure>



Obviously, the OpenShift Cluster Operator deployed CSI drivers, but no configuration was made for the velero-vsphere-plugin to point to a default secrets for velero backup-driver (different from standard CSI Installation )
So we need to create a secret from a vSphere config File  hosting the clusterID and the vSphere Credentials,
and apply a configmap to ppdm to use the secret.

Use the following example to create a config file with your vCenter csi account:

{% highlight bash %}
cat > csi-vsphere.conf <<EOF
[Global]
cluster-id = "$(oc get clusterversion -o jsonpath='{.items[].spec.clusterID}{"\n"}')"
[VirtualCenter "vcsa1.home.labbuildr.com"]
insecure-flag = "true"
user = <your vsphere csi user>
password = <your csi acccount secret>
port = 443
datacenters = "<your dc>"
EOF   
{% endhighlight %}

Create a Secret bfrom that file using:
{% highlight bash %}
oc -n velero-ppdm create secret generic velero-vsphere-config-secret --from-file=csi-vsphere.conf  
{% endhighlight %}

Apply the configmap for the velero vSphere Plugin
{% highlight bash %}
oc -n velero-ppdm apply -f - <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: velero-vsphere-plugin-config
  namespace: velero-ppdm
  labels:
    app.kubernetes.io/part-of: powerprotect.dell.com
data:
  cluster_flavor: VANILLA
  vsphere_secret_name: velero-vsphere-config-secret
  vsphere_secret_namespace: velero-ppdm
EOF
{% endhighlight %}

And delete the existing backup-driver pod
{% highlight bash %}
oc delete pod "$(oc -n velero-ppdm get pods | grep backup-driver | awk '{print $1}')" -n velero-ppdm
{% endhighlight %}

We can see the results with:
{% highlight bash %}
oc logs "$(oc -n velero-ppdm get pods | grep backup-driver | awk '{print $1}')" -n velero-ppdm
{% endhighlight %}
<figure class="full">
	<img src="/images/velero-backup-driver.png" alt="">
	<figcaption>Backup Driver Logs</figcaption>
</figure>


## 3. Run a backup !

Follow the Admin Guide to create a Protection Policy for your Discovered OpenShift Assets and select a Namespace with a PVC.
Run the Policy and examine the Status Details.
<figure class="full">
	<img src="/images/error_csi.png" alt="Error in Step Log">
	<figcaption>CSI error for FCD PVC</figcaption>
</figure>

Unfortunately, PPDM tries to trigger a CSI based Backup instead of a FCD based PVC Snapshot. This is because upon initial discovery, PPDM did not correctly identify a "VANILLA_VSPHERE" Cluster, as the cluster-csi-operator uses a different install location for the csi drivers. This only Occurs on 4.10 Fresh installs, not on updates from 4.9 to 4.10 !


What we need to do now, is patching the Inventory Source of Our OpenShift Cluster in PPDM using the REST API.

## 4. Patching a PPDM Inventory Source for fresh deployed 4.10 Clusters

### 4.1 Get and use the bearer token for subsequent requests
{% highlight bash %}
{% raw %}      
PPDM_SERVER=<your ppdm fqdn>
PPDM_USERNAME=<your ppdm username>
PPDM_PASSWORD=<your ppdm password>
TOKEN=$(curl -k --request POST \
  --url "https://${PPDM_SERVER}:8443/api/v2/login" \
  --header 'content-type: application/json' \
  --data '{"username":"'${PPDM_USERNAME}'","password":"'${PPDM_PASSWORD}'"}' | jq -r .access_token)
{% endraw %}      
{% endhighlight %}
### 4.2 Get the inventory id´s for vCenter and the OpenShift cluster
{% highlight bash %}
{% raw %}      
K8S_INVENTORY_ID=$(curl -k --request GET "https://${PPDM_SERVER}:8443/api/v2/inventory-sources" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer ${TOKEN}" \
| jq '[.content[] | select(.address=="api.ocs1.home.labbuildr.com")]| .[].id' -r)

VCENTER_ID=$(curl -k --request GET "https://${PPDM_SERVER}:8443/api/v2/inventory-sources" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer ${TOKEN}" \
| jq '[.content[] | select(.address=="vcsa1.home.labbuildr.com")]| .[].id' -r)
{% endraw %}      
{% endhighlight %}
### 4.3 query the distribution type field of the kubernetes inventory source by using above K8S_INVENTORY_ID

{% highlight bash %}
{% raw %} 
curl -k --request GET "https://${PPDM_SERVER}:8443/api/v2/inventory-sources/$K8S_INVENTORY_ID" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer ${TOKEN}" | jq .details.k8s.distributionType
{% endraw %} 
{% endhighlight %}
If the Distribution type shows NON_VSPHERE, we need to patch it
<figure class="full">
	<img src="/images/no_vsphere.png" alt="Error in Step Log">
	<figcaption>Wrong Distribution Type</figcaption>
</figure>

### 4.4 Read the inventory into variable:

{% highlight bash %}
{% raw %} 
INVENTORY_SOURCE=$(curl -k --request GET "https://${PPDM_SERVER}:8443/api/v2/inventory-sources/$K8S_INVENTORY_ID" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer ${TOKEN}")
{% endraw %} 
{% endhighlight %}
### 4.5 Patch the variable content using jq:

{% highlight bash %}
{% raw %} 
INVENTORY_SOURCE=$(echo $INVENTORY_SOURCE \
| jq --arg distributionType "VANILLA_ON_VSPHERE" '.details.k8s.distributionType |= $distributionType')
INVENTORY_SOURCE=$(echo $INVENTORY_SOURCE \
| jq --arg vCenterId "$VCENTER_ID" '.details.k8s.vCenterId |= $vCenterId')
{% endraw %} 
{% endhighlight %}
### 4.6 validate the variable using jq:

{% highlight bash %}
{% raw %} 
echo $INVENTORY_SOURCE| jq .details.k8s
{% endraw %} 
{% endhighlight %}
<figure class="full">
	<img src="/images/vanilla.png" alt="Error in Step Log">
	<figcaption>Correct Distribution Type</figcaption>
</figure>
### 4.7 And Patch that PPDM !!!

{% highlight bash %}
curl -k -X PUT "https://${PPDM_SERVER}:8443/api/v2/inventory-sources/$K8S_INVENTORY_ID" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
-d "$INVENTORY_SOURCE"
{% endhighlight %}

## 5. Run the Protection Policy Again

In the Job List in PPDM, select the failed job and click on restart
<figure class="full">
	<img src="/images/restart_pp.png" alt="Error in Step Log">
	<figcaption>Restart Job</figcaption>
</figure>

The Job should now run Successful, and take PVC Snapshots (using First Class Disks) instead of CSI Snapshots
<figure class="full">
	<img src="/images/successfull_fcd.png" alt="Successful Job ">
	<figcaption>Successful Job</figcaption>
</figure>

