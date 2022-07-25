---
layout: post
title: "Customizing PPDM Pods to use Node Affinity by patching K8S Inventory Source"
description: "Tipps and Troubleshooting"
modified: 2022-07-23
comments: true
published: true
tags: [OpenShift, json, rest, PowerProtect, kubernetes, k8s]
image:
  path: /images/InventorySourceK8S.png
  feature: InventorySourceK8S.png
  credit: 
  creditlink: 
---
### Create Customized POD Configs for PPDM Pod´s
*Disclaimer: use this at your own Risk*  
<figure class="full">
	<img src="/images/parental_advisory.png" alt="">
	<figcaption>Warning, Contains JSON and YAML !!!</figcaption>
</figure>
## why that ?
The PowerProtect DataManager Inventory Source uses Standardized Configurations and Configmaps for the Pods deployed by PPDM, e.gh. cProxy, PowerProtect Controller, as well as Velero.

In this Post, i will use an example to deploy the cProxies to dedicated nodes using [Node Affinity](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/). This makes Perfect sense if you want to separate Backup from Production Nodes.

More Examples could include using cni plugins like multus, dns configurations etc.

The Method described here is available from PPDM 19.10 and will be surfaced to the UI in Future Versions 

## 1. what we need
The below examples must be run from a bash shell. We will use jq to modify json Documents.

## 2. Adding labels to worker nodes
in order to use Node Affinity for our Pods, we first need to label nodes for dedicated usage.

In this Example we label a node with *tier=backup*

A corresponding Pod Configuration example would look like:

{% highlight yaml %}
apiVersion: apps/v1
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tier
            operator: In
            values:
            - backup            
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
{% endhighlight %}
We will use a customized template section later for our cProxy

So first, tag the Node(s) you want to use for Backup:

{% highlight shell %}
kubectl label nodes ocpcluster1-ntsgq-worker-local-2xl2z tier=backup
{% endhighlight %}


## 2. Create a Configuration Patch for the cProxy
We create manifest Patch for the cProxy from a yaml Document.
This will be base64 encoded and presented as a Value to POD_CONFIG type to our Inventory Source
The [API Reference](https://developer.dell.com/apis/4378/versions/19.11/reference/ppdm-public.oas2.yaml/components/schemas/InventorySourceK8s) describes the Format of the Configuration.

The *CPROXY_CONFIG* variable below will contain the base64 Document

{% highlight shell %}
CPROXY_CONFIG=$(base64 -w0 <<EOF
---
metadata:
  labels:
    app: cproxy
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tier
            operator: In
            values:
            - backup      
EOF
)    
{% endhighlight %}

## 3. Patching the inventory Source using the PowerProtect Datamanager API
You might want to review the [PowerProtect Datamanager API Documentation](https://developer.dell.com/apis/4378/versions/19.11/docs/introduction.md)

for the Following Commands, we will leverage some BASH Variables to specify your environment:
{% highlight shell %}
PPDM_SERVER=<your ppdm fqdn>
PPDM_USERNAME=<your ppdm username>
PPDM_PASSWORD=<your ppdm password>
K8S_ADDRESS=<your k8s api address the cluster is registered with, see UI , Asset Sources -->, Kubernetes--> Address>
{% endhighlight %}

### 3.1 Logging in to the API and retrieve the Bearer Token
The below code will read the Bearer Token into the *TOKEN* variable

{% highlight shell %}
TOKEN=$(curl -k --request POST \
  --url https://${PPDM_SERVER}:8443/api/v2/login \
  --header 'content-type: application/json' \
  --data '{"username":"'${PPDM_USERNAME}'","password":"'${PPDM_PASSWORD}'"}' | jq -r .access_token)
{% endhighlight %}


### 3.2 Select the Inventory Source ID based on the Asset Source Address 

Select inventory ID matching your Asset Source Address :
{% highlight shell %}
K8S_INVENTORY_ID=$(curl -k --request GET https://${PPDM_SERVER}:8443/api/v2/inventory-sources \
--header "Content-Type: application/json" \
--header "Authorization: Bearer ${TOKEN}" \
| jq --arg k8saddress "${K8S_ADDRESS}" '[.content[] | select(.address==$k8saddress)]| .[].id' -r)
{% endhighlight %}

### 3.3 Read Inventory Source into Variable

With the *K8S_INVENTORY_ID* from above, we read the Inventory Source JSON Document into a Variable

{% highlight shell %}
INVENTORY_SOURCE=$(curl -k --request GET https://${PPDM_SERVER}:8443/api/v2/inventory-sources/$K8S_INVENTORY_ID \
--header "Content-Type: application/json" \
--header "Authorization: Bearer ${TOKEN}")
{% endhighlight %}


### 3.4 Adding the Patched cProxy Config to the Variable

Using jq, we will modify the Inventory Source JSON Document to include our base64 cproxy Config.

for that, we will add an list with content 

{% highlight YAML %}
"configurations": [
        {
          "type": "POD_CONFIG",
          "key": "CPROXY",
          "value": "someBase64Document"
        }
      ]
{% endhighlight %}
Other *POD_CONFIGS* key´s we could modify are *POWERPROTECT_CONTROLLER* and *VELERO*
the below json code will take care for this:

{% highlight shell %}
INVENTORY_SOURCE=$(echo $INVENTORY_SOURCE| \
 jq --arg cproxyConfig "${CPROXY_CONFIG}" '.details.k8s.configurations += [{"type": "POD_CONFIG","key": "CPROXY", "value": $cproxyConfig}]')
{% endhighlight %}

### 3.4 Patching the Inventory Source in PPDM

We now use a POST request to upload the Patched inventory Source Document to PPDM

{% highlight shell %}
curl -k -X PUT https://${PPDM_SERVER}:8443/api/v2/inventory-sources/$K8S_INVENTORY_ID \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
-d "$INVENTORY_SOURCE"
{% endhighlight %}

We can verify the patch by checking the PowerProtect Configmap in the PPDM Namespace.

{% highlight shell %}
kubectl get configmap  ppdm-controller-config -n powerprotect -o=yaml
{% endhighlight %}


The configmap must now contain *cproxy-pod-custom-config* in data

<figure class="full">
	<img src="/images/patched-cproxy.png" alt="cproxy-pod-custom-config">
	<figcaption>cproxy-pod-custom-config</figcaption>
</figure>


The next Backup job will now use the tagged node(s) for Backup !

### 3.5 Start a Backup and verify the node affinity for cproxy pod

First, identify the node you labeled

{% highlight shell %}
kubectl get nodes -l 'tier in (backup)'
{% endhighlight %}

once the backup job creates the cproxy, this should be done on one of the identified nodes:
{% highlight shell %}
kubectl get pods -n powerprotect -o wide
{% endhighlight %}



<figure class="full">
	<img src="/images/validate_pod_affinity.png" alt="cproxy-pod-custom-config">
	<figcaption>validate_pod_affinity</figcaption>
</figure>


This concludes the Patching of cproxy COnfiguration, stay tuned for more