# AKS Cluster Setup
This guide walks through the steps to install Calico policy to Azure AKS clusters - Calcio OSS (Project Calico) is not covered in this guide; follow the [Project Calico docs](https://docs.projectcalico.org/getting-started/kubernetes/managed-public-cloud/aks) for further instructions. Note Azure CNI is the only supported CNI for Calico; therefore, only Calico Policy can be deployed.
Commercial Calico offerings:
1. Calico Cloud
2. Calico Enterprise

## Azure Resource and AKS configuration
To setup the Azure subscription, resources and AKS cluster follow the [Azure Account Setup](https://github.com/chrisman-io/AKS-cluster-setup/tree/main/Azure-Account-Setup) guide.

## Calico Cloud
Once logged into the [Calico Cloud](https://www.calicocloud.io/home) portal follow the guided instructions to connect a cluster and run the installation shell script. The script will check if the AKS cluster is compatiable.

## Calico Enterprise
This guide summerizes the installation steps in the [Calico Enterprise](https://docs.tigera.io/) documentation.

1. Configure a storage class
The following configuration provisions a storage class and Calico Enterprise to use LRS disks for log storage
```yaml
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tigera-elasticsearch
provisioner: kubernetes.io/azure-file
mountOptions:
  - uid=1000
  - gid=1000
  - mfsymlinks
  - nobrl
  - cache=none
parameters:
  skuName: Standard_LRS
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```
2. Install the Tigera operator and custom resource definitions.
```
kubectl create -f https://docs.tigera.io/manifests/tigera-operator.yaml
```
3. Install the Prometheus operator and related custom resource definitions. The Prometheus operator will be used to deploy Prometheus server and Alertmanager to monitor Calico Enterprise metrics.
```
kubectl create -f https://docs.tigera.io/manifests/tigera-prometheus-operator.yaml
```
4. Install your pull secret
The pull secret json file is required from Tigera in order to pull the required images from Tigera's registry. The json file must be accessible from cloud shell - use the file upload operation if the file can be transferred to the Cloud Shell platform.
```
kubectl create secret generic tigera-pull-secret \
    --type=kubernetes.io/dockerconfigjson -n tigera-operator \
    --from-file=.dockerconfigjson=<path/to/pull/secret>
```
5. Install the Tigera custom resources
```
kubectl create -f https://docs.tigera.io/manifests/aks/custom-resources.yaml
```
You can now monitor progress with the following command:
```
watch kubectl get tigerastatus
```
The `apiserver` must be `Available` to proceed.
6. Install the Calico Enterprise license
The `license.yaml` file is provided by Tigera and must be accessible from the Cloud Shell.
```
kubectl create -f </path/to/license.yaml>
```
Monitor and wait for all the Calico components to be ready:
```
watch kubectl get tigerastatus
```
7. Secure Calico Enterprise with network policy
Network policies are configured to secure the Calico platform. This creates a Network Tier called `allow-tigera` and deploys a number of policies
```
kubectl create -f https://docs.tigera.io/manifests/tigera-policies-managed.yaml
```
8. The Tigera Manager web UI by default in not accessible from outside the K8s cluster. Deploying a Load Balancer resource enables AKS to provision an Azure loadbalancer with a Public IP
```yaml
# Enabling AKS to deploy an Azure loadbalancer
kubectl apply -f - <<EOF
kind: Service
apiVersion: v1
metadata:
  name: tigera-manager-external
  namespace: tigera-manager
spec:
  type: LoadBalancer
  selector:
    k8s-app: tigera-manager
  externalTrafficPolicy: Local
  ports:
  - port: 9443
    targetPort: 9443
    protocol: TCP
EOF
```
Obtain the external IP of the service:
```
# Retrieve the external IP of the Azure Loadbalancer VIP for Tigera Manager Web UI
kubectl get services -n tigera-manager
```
The web UI can now be accessed via `https://<external IP>:9443`
<br>

9. Obtain the Web UI token and Kibana password
```
# Default Calico configured administrator username is Jane. Obtain the token for Jane
kubectl get secret $(kubectl get serviceaccount jane -o jsonpath='{range .secrets[*]}{.name}{"\n"}{end}' | grep token) -o go-template='{{.data.token | base64decode}}'
```
Copy the token in order to log into the web UI.
<br>

```
# Obtain the Kibana password for elastic user
kubectl -n tigera-elasticsearch get secret tigera-secure-es-elastic-user -o go-template='{{.data.elastic | base64decode}}'
```
Copy the password and use `elastic` as the username to access Kibana (launched form the web UI)
