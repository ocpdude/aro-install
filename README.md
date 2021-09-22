# Azure Red Hat OpenShift (ARO) Installation
Azure Red Hat OpenShift is jointly engineered, operated, and supported Kubernetes Platform by Red Hat and Microsoft to provide an integrated support experience. There are no virtual machines to operate, and no patching is required. Master, infrastructure, and application nodes are patched, updated, and monitored on your behalf by Red Hat and Microsoft. Your Azure Red Hat OpenShift clusters are deployed into your Azure subscription and are included on your Azure bill.

You can choose your own registry, networking, storage, and CI/CD solutions, or use the built-in solutions for automated source code management, container and application builds, deployments, scaling, health management, and more. Azure Red Hat OpenShift provides an integrated sign-on experience through Azure Active Directory.

Ref: Microsoft ARO Tutorial - https://docs.microsoft.com/en-us/azure/openshift/tutorial-create-cluster

Azure Red Hat OpenShift requires a minimum of 40 cores to create and run an OpenShift cluster. The default Azure resource quota for a new Azure subscription does not meet this requirement. To request an increase in your resource limit.

Ref: Standard quota: Increase limits by VM series - https://docs.microsoft.com/en-us/azure/azure-portal/supportability/per-vm-quota-requests

### Prereq's for this script
1. Azure CLI - https://docs.microsoft.com/en-us/cli/azure/
2. jq binary - https://stedolan.github.io/jq/download/
3. oc / kubectl binaries - https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/
4. Red Hat Pull Secret - https://cloud.redhat.com 
* If you do not use a pull-secret, you can still create ARO, but access to some resources will be restricted.

### DNS
There are two ways to install ARO, with your own DNS domain or with one randomly generated as `<random>.<location>.aroapp.io`. If you wish to use your own domain, during the install you'll pass the --domain example.com flag. After the install, you'll need to query the IP assignments and input those into your DNS domain as "A" records for `api.<cluster>.<domain.com>` and `*.apps.<cluster>.<domain.com>`.

#### Query: `az aro show -n -g --query '{api:apiserverProfile.ip, ingress:ingressProfiles[0].ip}'`


----
Environment Variables to simplfy the installation tasks: Modify the script below for your purposes... note the network settings for the VNET & Subs.
```
export LOCATION=<Azure region>

export RESOURCEGROUP=<provisioned_rg>

export CLUSTERRG=<dynamic_cluster_rg>

export CLUSTERNAME=<ocp_clustername>

export NETWORKRG=<provisioned_vnet_rg>

export VNET=<provisioned_vnet_name>

export MASTSUB=<master_subnet_name>

export WORKSUB=<worker_subnet_name>

export SP_ID=<service_principal_name>

export SUBSCRIPTION=<xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx>
```

Set the subcription to use (if needed)

`az account set --subscription $SUBSCRIPTION`

Register resource providers for the subscription

```
az provider register -n Microsoft.RedHatOpenShift --wait

az provider register -n Microsoft.Compute --wait

az provider register -n Microsoft.Storage --wait

az provider register -n Microsoft.Authorization --wait
```

Create your primary resource-group

`az group create --name $RESOURCEGROUP --location $LOCATION`

Create your network-resource-group

`az group create --name $NETWORKRG --location $LOCATION`

Set your VNET CIDR

`az network vnet create --resource-group $NETWORKRG --name $VNET --address-prefixes 10.0.0.0/22`

Configure your network-subnets

`az network vnet subnet create  --resource-group $NETWORKRG --vnet-name $VNET --name $MASTSUB --address-prefixes 10.0.0.0/24 --service-endpoints Microsoft.ContainerRegistry`

`az network vnet subnet create  --resource-group $NETWORKRG --vnet-name $VNET --name $WORKSUB --address-prefixes 10.0.2.0/24 --service-endpoints Microsoft.ContainerRegistry`

`az network vnet subnet update  --name $MASTSUB --resource-group $NETWORKRG  --vnet-name $VNET  --disable-private-link-service-network-policies true`

`az network vnet subnet update  --name $WORKSUB --resource-group $NETWORKRG  --vnet-name $VNET  --disable-private-link-service-network-policies true`

Set the Contributor Role on the Service Principal for each resource-group

`az ad sp create-for-rbac --role "Contributor" --name $SP_ID --scopes /subscriptions/$SUBSCRIPTION /subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP  /subscriptions/$SUBSCRIPTION/resourceGroups/$NETWORKRG`

Set the User Access Administrator Role on the Service Principal for each resource-group

`az ad sp create-for-rbac --role "User Access Administrator" --name $SP_ID --scopes /subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP /subscriptions/$SUBSCRIPTION/resourceGroups/$NETWORKRG`

Set the Network Contributor Role on the Service Principal for the VNET

`az ad sp create-for-rbac --role "Network Contributor" --name $SP_ID --scopes /subscriptions/$SUBSCRIPTION/resourceGroups/$NETWORKRG/providers/Microsoft.Network/virtualNetworks/$VNET`

Pull your Service Principal ID & Password

`az ad sp create-for-rbac --name $SP_ID --role Contributor > serviceprincipal.json`

```
export SP_APPID=$(jq -r .appId serviceprincipal.json)

export SP_PASSWD=$(jq -r '.password' serviceprincipal.json)
```

Download your pull-secret from https://cloud.redhat.com and save it as 'pull-secret.txt'

Build your ARO using the Service Principal account

`az aro create --resource-group $RESOURCEGROUP --cluster-resource-group $CLUSTERRG --name $CLUSTERNAME --vnet-resource-group $NETWORKRG --vnet $VNET  --client-id $SP_APPID --client-secret $SP_PASSWD --master-subnet $MASTSUB  --worker-subnet $WORKSUB --pull-secret @pull-secret.txt`

---
After the install, fetch your credentials & console URL

`az aro list-credentials --name $CLUSTERNAME --resource-group $RESOURCEGROUP`

It should look like this...
```
{
  "kubeadminPassword": "<auto_generated_password>",
  "kubeadminUsername": "kubeadmin"
}
```

`az aro show --name $CLUSTERNAME --resource-group $RESOURCEGROUP --query "consoleProfile.url" -o tsv`

---
Access ARO using the CLI

`apiServer=$(az aro show -g $RESOURCEGROUP -n $CLUSTERNAME --query apiserverProfile.url -o tsv)`

`oc login $apiServer -u kubeadmin -p <kubeadmin_password>`

---
Deleting Your Cluster

To delete your cluster just pass the `az aro delete`, this will remove all dynamicly generated content through the `az aro` build strips above. The primary resource-group, network resrouce-group, vnet and subs will remain.

`az aro delete --resource-group $RESOURCEGROUP --name $CLUSTERNAME`