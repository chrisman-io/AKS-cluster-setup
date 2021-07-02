# AKS Account Setup
Follow these steps if you need to setup your Azure account and environment

## Prerequisites
You will need to have an active Azure account with a valid subscription and have a role that enables creation of Azure resource. Recognize creating the AKS environment will incur running costs

## Instructions
These instructions will use a browser based method using cloud shell

1. Login to Azure portal http://portal.azure.com  
2. Launch a Cloud Shell session and choose Bash Shell
3. Ensure your account deploys into the correct subscription
```
# View subscriptions
az account list
```
```
Verify subscription
az account show
```
```
# Set correct subscription (if needed)
az account set --subscription <subscription_id>

# Verify correct subscription is now set
az account show
```
4. Create Azure Service Principal to use
```
az ad sp create-for-rbac --skip-assignment
```
**Note the output as these are required and only printed once!**  
It will be similar to:
```
"appId": "7248f250-0000-0000-0000-dbdeb8400d85",
"displayName": "azure-cli-2017-10-15-02-20-15",
"name": "http://azure-cli-2017-10-15-02-20-15",
"password": "77851d2c-0000-0000-0000-cb3ebc97975a",
"tenant": "72f988bf-0000-0000-0000-2d7cd011db47"
```
5. Set local variables from the output in step 4.
```
# Persist for Later Sessions in Case of Timeout
APPID=<appId>
echo export APPID=$APPID >> ~/.bashrc
CLIENTSECRET=<password>
echo export CLIENTSECRET=$CLIENTSECRET >> ~/.bashrc
```
6. Create Resource Group and set region
```
# Set Resource Group Name using the unique suffix
RGNAME=<RG Name>
# Persist for Later Sessions in Case of Timeout
echo export RGNAME=$RGNAME >> ~/.bashrc
# Set Region (Location)
LOCATION=<Azure region>
# Persist for Later Sessions in Case of Timeout
echo export LOCATION=<Azure region> >> ~/.bashrc
# Create Resource Group
az group create -n $RGNAME -l $LOCATION
```
7. Set Cluster Name
```
# Set AKS Cluster Name
CLUSTERNAME=<Cluster Name>
# Look at AKS Cluster Name for Future Reference
echo $CLUSTERNAME
# Persist for Later Sessions in Case of Timeout
echo export CLUSTERNAME=<Cluster Name> >> ~/.bashrc
```
8. Get available Kubernetes versions for the region and set version
```
az aks get-versions -l $LOCATION --output table

KubernetesVersion    Upgrades
-------------------  -----------------------------------------
1.19.3(preview)      None available
1.19.0(preview)      1.19.3(preview)
1.18.10              1.19.0(preview), 1.19.3(preview)
1.18.8               1.18.10, 1.19.0(preview), 1.19.3(preview)
1.17.13              1.18.8, 1.18.10
1.17.11              1.17.13, 1.18.8, 1.18.10
```
Set K8s version
```
K8SVERSION=1.19.0
```
9. Create cluster with required Node Count and set Azure CNI which is required for Calico
```
# Create AKS Cluster
az aks create -n $CLUSTERNAME -g $RGNAME \
--kubernetes-version $K8SVERSION \
--service-principal $APPID \
--client-secret $CLIENTSECRET \
--generate-ssh-keys -l $LOCATION \
--node-count 3 \
--network-plugin azure \
--no-wait
```
10. Provisioning the clsuter can take up to 10-15 mins. Verify when the cluster has been successfully created. Ensure `ProvisionState` is `Succeeded`
```
az aks list -o table
```
11. Get the Kubernetes config files for the new AKS cluster. This will overwrite to $HOME/.kube/config
```
az aks get-credentials -n $CLUSTERNAME -g $RGNAME
```
12. Export the `Kubeconfig` file 
```
export KUBECONFIG=:$HOME/.kube/config
```

