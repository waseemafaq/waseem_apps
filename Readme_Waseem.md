# Azure AGIC Install using Add-On on Azure AKS Cluster (Greenfield)

## Step-01 Set the subscription 
```t
$subscription = "172111df-eb44-45a1-b46d-a5a458c77065" # Personal
az account set --subscription $subscription

```

## Step-02: Create AKS Cluster
```t
# Create Resource Group
az group create --location eastus --resource-group agicdemo

# Create AKS Cluser
az aks create --name agic-cluster `
              --resource-group agicdemo `
              --network-plugin azure `
              --node-vm-size Standard_B2ms `
              --enable-managed-identity `
              --enable-addons ingress-appgw `
              --appgw-name agic-appgw `
              --appgw-subnet-cidr "10.225.0.0/16" `
              --node-count 1 `
              --generate-ssh-keys 
```

## Step-03: Verify AKS Add On 
```t
# List Kubernetes Deployments in kube-system namespace
az account set --subscription 172111df-eb44-45a1-b46d-a5a458c77065
az aks get-credentials --resource-group agicdemo --name agic-cluster --overwrite-existing

kubectl get deploy -n kube-system

Observation:
1. Should find the deployment with name "ingress-appgw-deployment"
2. This is the Azure Application Gateway Ingress Controller Kubernetes Deployment Object

# List Pods
kubectl get pods -n kube-system

# Describe Pod
kubectl -n kube-system describe pod <AGIC-POD-NAME>
kubectl -n kube-system describe pod ingress-appgw-deployment-5948b5c7c6-bxtnr
Observation:
1. Review the line where you can find ingress current version downloaded
2. Pulling image "mcr.microsoft.com/azure-application-gateway/kubernetes-ingress:1.5.3"
3. You can also run the below command to find the AppGW Ingress version
kubectl get deploy ingress-appgw-deployment -o yaml -n kube-system | select-string "image:"

# Verify ingress-appgw pod Logs
kubectl -n kube-system logs -f $(kubectl -n kube-system get po | egrep -o 'ingress-appgw-deployment[A-Za-z0-9-]+')
```

## Step-04: For cost saving , change instance count to 1 and save
```t
    Go to portal > under configuration > instance count 1

# Stop Azure AKS and Application Gateway

# Azure AKS Cluster
In portal, go to Overview Tab and click on "STOP"

# Azure Application Gateway STOP
az network application-gateway stop --name <APPGW-NAME> --resource-group <RESOURCE-GROUP>
az network application-gateway stop --name agic-appgw --resource-group MC_agicdemo_agic-cluster_eastus

```

## Step-05: Start Azure AKS and Application Gateway
```t
# Important Note
- Wait for 15 minutes before starting them back

# Azure AKS Cluster
In portal, go to Overview Tab and click on "START"

# Azure Application Gateway START
az network application-gateway start --name <APPGW-NAME> --resource-group <RESOURCE-GROUP>
az network application-gateway start --name agic-appgw --resource-group MC_agicdemo_agic-cluster_eastus

Get-AzApplicationGateway -Name "agic-appgw" -ResourceGroupName "MC_agicdemo_agic-cluster_eastus"

```


## Additional References
- [Greenfield: AGIC Add-On on AKS Cluster](https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-new)
- [Brownfield: AGIC Add-On on AKS Cluster](https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-existing)
- [Greenfield: AGIC using Helm on AKS Cluster](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-install-new)
- [Brownfield: AGIC using Helm on AKS Cluster](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-install-existing)
- https://azure.microsoft.com/en-us/blog/application-gateway-ingress-controller-for-azure-kubernetes-service/
- https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-new?tryIt=true&source=docs#deploy-an-aks-cluster-with-the-add-on-enabled  
- https://learn.microsoft.com/en-us/azure/architecture/example-scenario/aks-agic/aks-agic



===================================================================================================================================================================================

# Delegate Domain to Azure DNS

## Step-01: DNS Zones - Create DNS Zone
- Go to Service -> **DNS Zones**
- **Subscription:** Az-Learn-Sub
- **Resource Group:** dns-zones
- **Name:** infrainsight.in
- **Resource Group Location:** East US
- Click on **Review + Create**

## Step-02: Make a note of Azure Nameservers
- Go to Services -> **DNS Zones** -> **infrainsight.in**
- Make a note of Nameservers

ns1-04.azure-dns.com
ns2-04.azure-dns.net
ns3-04.azure-dns.org
ns4-04.azure-dns.info


## Step-03: Update Nameservers at your Domain provider (Mine is godaddy)
- **Verify before updation**
```t
nslookup -type=SOA infrainsight.in
nslookup -type=NS infrainsight.in

nslookup -type=SOA infrainsight.in 8.8.8.8
nslookup -type=NS infrainsight.in 8.8.8.8
```


===================================================================================================================================================================================

# Kubernetes ExternalDNS to create Record Sets in Azure DNS from AKS

## Step-01: Create External DNS Manifests
- External-DNS needs permissions to Azure DNS to modify (Add, Update, Delete DNS Record Sets)
- We can provide permissions to External-DNS pod in two ways in Azure 
  - Using Azure Service Principal
  - Managed identity using AKS Kubelet identity
- We are going to use `Managed Identity` for providing necessary permissions here which is latest and greatest in Azure as on today. 

## Step-02: Fetching the Kubelet identity
```t
# Option-1: Get PRINCIPAL_ID using Commands
$CLUSTER_GROUP="agicdemo"
$CLUSTERNAME= "agic-cluster"
$PRINCIPAL_ID= $(az aks show --resource-group $CLUSTER_GROUP --name $CLUSTERNAME --query "identityProfile.kubeletidentity.objectId" --output tsv)
$PRINCIPAL_ID 

# Option-2: Get PRINCIPAL_ID using Azure Portal
1. Go to Resource Groups -> MC_agicdemo_agic-cluster_eastus
2. Find Managed Identity Resource with name "agic-cluster-agentpool"
3. Make a note of "Object (principal) ID"
PRINCIPAL_ID=5657b353-282c-4208-93a0-c99ac23a4ef3
echo $PRINCIPAL_ID 
```

## Step-03: Assign rights for the Kubelet identity
- Grant access to Azure DNS zone for the kublet identity.
```t
## Option-1: Get DNS_ID using Command Line
# DNS zone name like example.com or sub.example.com
$AZURE_DNS_ZONE="infrainsight.in" 

# resource group where DNS zone is hosted
$AZURE_DNS_ZONE_RESOURCE_GROUP="dns-zones" 

# Option-1: fetch DNS id used to grant access to the kublet identity
$DNS_ID=$(az network dns zone show --name $AZURE_DNS_ZONE --resource-group $AZURE_DNS_ZONE_RESOURCE_GROUP --query "id" --output tsv)
$DNS_ID

## Option-2: Get DNS_ID using Azure Portal
1. Go to Azure Portal -> DNS Zones -> YOURDOMAIN.COM (infrainsight.in)
2. Go to Properties
3. Make a note of "Resource ID"
DNS_ID=/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/dns-zones/providers/Microsoft.Network/dnszones/infrainsight.in
echo $DNS_ID

# Grant access to Azure DNS zone for the kubelet identity.
az role assignment create --role "DNS Zone Contributor" --assignee $PRINCIPAL_ID --scope $DNS_ID

# Verify the Azure Role Assignment in Managed Identities
1. Go to Azure Portal -> Managed Identities -> agic-cluster-agentpool
2. Go to Azur role assignments Tab
3. We can see "DNS Zone Contributor" role assigned to "agic-cluster-agentpool" Managed Identity
```

## Step-04: Gather Information Required for azure.json file
```t
# To get Azure Tenant ID
az account show --query "tenantId"

# To get Azure Subscription ID
az account show --query "id"
```

## Step-05: Make a note of Client Id and update in azure.json
- Go to managed Identities -> find the Agent Pool MI (agic-cluster-agentpool) -> Click on it
- Go to **Overview** -> Make a note of **Client ID**
- Update in **azure.json** value for **userAssignedIdentityID**
```t
# Get value for userAssignedIdentityID
  "userAssignedIdentityID": "b1a0abb5-6c7d-45cc-bbc8-5f89fe0aa50f"
```

## Step-06: Use the azure.json file to create a Kubernetes secret:
```t
# List k8s secrets
kubectl get secrets

# Create k8s secret with azure.json
kubectl create secret generic azure-config-file --namespace "default" --from-file azure.json

# List k8s secrets
kubectl get secrets

# k8s secret output as yaml
kubectl get secret azure-config-file -o yaml
Observation:
1. You should see a base64 encoded value
2. Decode it with https://www.base64decode.org/ to review it
```
 
## Step-07: Deploy ExternalDNS
```t
# Deploy ExternalDNS 
kubectl apply -f external-dns.yaml

# Verify ExternalDNS Logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

```t
# Error Type: 400
time="2020-08-24T11:25:04Z" level=error msg="azure.BearerAuthorizer#WithAuthorization: Failed to refresh the Token for request to https://management.azure.com/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/dns-zones/providers/Microsoft.Network/dnsZones?api-version=2018-05-01: StatusCode=400 -- Original Error: adal: Refresh request failed. Status Code = '400'. Response body: {\"error\":\"invalid_request\",\"error_description\":\"Identity not found\"}"

# Error Type: 403
Notes: Error 403 will come when our Managed Service Identity dont have access to respective destination resource 

# When all good, we should get log as below
time="2023-09-08T01:23:32Z" level=info msg="Instantiating new Kubernetes client"
time="2023-09-08T01:23:32Z" level=info msg="Using inCluster-config based on serviceaccount-token"
time="2023-09-08T01:23:32Z" level=info msg="Created Kubernetes client https://10.0.0.1:443"
time="2023-09-08T01:23:33Z" level=info msg="Using managed identity extension to retrieve access token for Azure API."
time="2023-09-08T01:23:33Z" level=info msg="Resolving to user assigned identity, client id is b1a0abb5-6c7d-45cc-bbc8-5f89fe0aa50f."
```

## External DNS References
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md
- https://github.com/kubernetes-sigs/external-dns
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md

===================================================================================================================================================================================

# AGIC Ingress - SSL using Lets Encrypt

## Step-01: Install Cert Manager using Helm
```t
# Install Helm CLI
https://helm.sh/docs/intro/install/

## For Windows install by downloading Package
https://github.com/stacksimplify/helm-masterclass/tree/main/01-Install-Docker-Desktop-and-HelmCLI#step-08-windows-install-helm-cli-using-package

# Verify Helm Version
helm version

# Create Namespace 
kubectl create namespace cert-manager

# Label the  cert-manager namespace to disable resource validation
kubectl label namespace cert-manager cert-manager.io/disable-validation=true

# Review Releases and Update latest release number below
https://github.com/cert-manager/cert-manager/releases/ 

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# List Helm Repos
helm repo list

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart (using this one)
helm install `
  cert-manager jetstack/cert-manager `
  --namespace cert-manager `
  --version v1.15.3 `
  --set installCRDs=true

# List Helm Releases in cert-manager namespace
helm list -n cert-manager

# helm status in cert-manager namespace
helm status cert-manager --show-resources -n cert-manager
Obserevation: 
1. Review all the resources created as part of this Helm Release

# Verify Cert Manager pods in cert-manager namespace
kubectl get pods --namespace cert-manager

# Verify Cert Manager Services
kubectl get svc --namespace cert-manager

# Verify All Kubernetes Resources created in cert-manager namespace
kubectl get all --namespace cert-manager


# To Install CRDs manually without HELM
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.crds.yaml

# To Uninstall CRDs manually
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.crds.yaml
```

### Step-02: Deploy Cluster Issuer
```t
# Deploy Cluster Issuer
kubectl apply -f cluster-issuer.yml

# List Cluster Issuer
kubectl get clusterissuer

# Describe Cluster Issuer
kubectl describe clusterissuer letsencrypt
```

## Step-03: Review Application NginxApp1,2 K8S Manifests
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml
- 03-NginxApp2-Deployment.yaml
- 04-NginxApp2-ClusterIP-Service.yaml


## Step-04 Deploy Kubernetes Manifests & Verify
- Certificate Request, Generation, Approval and Download and be ready might take from 1 hour to couple of days, if we make any mistakes it might also fail.
- For me it took, only 5 minutes to get the certificate from **https://letsencrypt.org/**
```t
# Verify if Cluster Issuer is already deployed
kubectl get clusterissuer

# Deploy Application
kubectl apply -f app-manifests

# Verify Pods
kubectl get pods

# Verify Kubernetes Secrets
kubectl get secrets

# YAML Output of Kubernetes Secrets
kubectl get secret sapp1-infrainsight-secret -o yaml
kubectl get secret sapp2-infrainsight-secret -o yaml
Observation:
1. Review tls.crt and tls.key

# Verify SSL Certificates (It should turn to True)
kubectl get certificate

## Sample Output
NAME                       READY   SECRET                     AGE
sapp1-infrainsight-secret   True    sapp1-infrainsight-secret   2m48s
sapp2-infrainsight-secret   True    sapp2-infrainsight-secret   2m48s

# Verify Cert Manager Pod Logs
kubectl -n cert-manager get pods 
kubectl -n cert-manager logs -f <cert-manager-POD-NAME>
kubectl -n cert-manager logs -f cert-manager-98c64c5bd-l5dbw


# Verify external-dns Controller logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
[or]
kubectl get pods
kubectl logs -f <External-DNS-Pod-Name>

# If NO External DNS Installed What we need to do ?
1. We need to manually add the DNS name in Azure DNS Zones before deploying "manifests"
2. Get the Public IP from Azure Application Gateway -> agic-appgw -> Overview Tab
3. Go to DNS Zones -> Add Record set 
sapp1.infrainsight.com
sapp2.infrainsight.com
```


## Step-07: Verify Settings in Azure Portal  for Azure Application Gateway
```t
# 1. AppGw Backend Pools
1. We should see App1 and App2 two backend pools created

# 2. AppGw Backend Settings
1. We should see App1 and App2 two backend settings created


# 3. AppGw Listeners
1. We should see two port 80 listeners created to redirect to port 443 for App1 and App2
2. We should see two port 443 listeners created to serve content for App1 and App2 with hostnames as sapp1.infrainsight.com, sapp2.infrainsight.com

# 4. AppGw Listener TLS Certificates 
1. We should see below listed two LetsEncrypt auto-uploaded to Application Gateway when we deployed our Ingress Manifest
1.1 cert-default-sapp1-infrainsight-secret
1.2 cert-default-sapp2-infrainsight-secret
2. This happened because we used "spec.tls.secretName" in our Ingress Service

# 5. AppGw Rules
1. Two Port 80 Rules: In Backend Targets
1.1 Target Type: Redirection
1.2 Redirection type: permanent
1.3 Redirection target: Listener
1.4 Target listener: Port 443 listener

2. Two Port 443 Rules created


# 6. AppGw Health Probes
1. Two HTTP Health probes created for App1 and App2
```


## Step-08: Access Application
```t
# Application HTTP URLs
http://sapp1.infrainsight.in/app1/index.html
http://sapp2.infrainsight.in/app2/index.html
Observation:
1. Should redirect to HTTPS url
2. We have added AGIC ssl-redirect annotation in Ingress Manifest

# Application HTTPS URLs
https://sapp1.infrainsight.in/app1/index.html
https://sapp2.infrainsight.in/app2/index.html
Observation
1. Review SSL Certificate from browser after accessing URL
2. We should valid SSL certificate generated by LetsEncrypt

```
## Step-09: Clean-Up
```t
# Delete Applications
kubectl delete -f app-manifests/
```

## Cert Manager References
- https://cert-manager.io/docs/installation/#default-static-install
- https://cert-manager.io/docs/installation/helm/
- https://docs.cert-manager.io/
- https://cert-manager.io/docs/installation/helm/#1-add-the-jetstack-helm-repository
- https://cert-manager.io/docs/configuration/
- https://cert-manager.io/docs/tutorials/acme/nginx-ingress/#step-6---configure-a-lets-encrypt-issuer
- https://letsencrypt.org/how-it-works/


===================================================================================================================================================================================

# Installing nginx ingress using HELM

## Step-01: Create Static Public IP
```t
# Get the resource group name of the AKS cluster 
az aks show --resource-group waseem-afaq-poc --name aks-01-waseem --query nodeResourceGroup -o tsv

# Create a public IP address with the static allocation
az network public-ip create --resource-group MC_waseem-afaq-poc_aks-01-waseem_eastus --name myAKSPublicIPForIngress --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv

```
- Make a note of Static IP which we will use in next step when installing Ingress Controller
```t
# Make a note of Public IP created for Ingress
172.190.158.154
```

## Step-02: Install Nginx Ingress Controller
- [ingress-nginx Controller Git Repo](https://github.com/kubernetes/ingress-nginx)
```t
# List Ingress Class
kubectl get ingressclass

# Install Helm3 (if not installed)
brew install helm

# Create a namespace for your ingress resources
kubectl create namespace ingress-nginx

# Add the official stable repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Search Helm Repo
helm search repo ingress-nginx
helm search repo ingress-nginx --versions

#  Customizing the Chart Before Installing. 
helm show values ingress-nginx/ingress-nginx

# Use Helm to deploy an NGINX ingress controller
helm upgrade ingress-nginx ingress-nginx/ingress-nginx `
    --namespace ingress-nginx `
    --set controller.replicaCount=2 `
    --set controller.service.externalTrafficPolicy=Local `
    --set controller.service.loadBalancerIP="172.190.158.154" 

# Uninstall
helm uninstall ingress-nginx --namespace ingress-nginx

# List Ingress Class
kubectl get ingressclass

# List Services with labels
kubectl get service -l app.kubernetes.io/name=ingress-nginx --namespace ingress-nginx
kubectl get service -n ingress-nginx

# List Pods
kubectl get pods -n ingress-nginx
kubectl get all -n ingress-nginx

# Helm List
helm list -n ingress-nginx

# Helm status --show-resources
helm status --show-resources <RELEASE-NAME> -n ingress-nginx
helm status --show-resources ingress-nginx -n ingress-nginx
Observation:
1. Will show all the kubernetes resources created as part of this Helm Chart in Helm Release


# Access Public IP
http://<Public-IP-created-for-Ingress>
http://20.121.21.149

# Output should be
404 Not Found from Nginx

# Verify Load Balancer on Azure Mgmt Console
Primarily refer Settings -> Frontend IP Configuration
```

## Step-04: Review Application k8s manifests with Ingress Class
### Step-04-01: kube-manifests folder: kube-manifests
#### Folder: 01-App1-NginxIC**
- 01-NginxApp1-Deployment.yml
- 02-NginxApp1-ClusterIP-Service.yml
- 03-Ingress.yml
```yaml
spec:
  ingressClassName: nginx 
```
#### Folder: 02-App2-AGIC
- 01-NginxApp2-Deployment.yml
- 02-NginxApp2-ClusterIP-Service.yml
- 03-Ingress.yml
```yaml
spec:
  ingressClassName: azure-application-gateway    
```

### Step-04-02: Deploy Application k8s manifests and verify
```t
# Deploy
kubectl apply -R -f kube-manifests

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress
kubectl get ingress

# Access Applications
http://<NGINXIC-IP>/app1/index.html
http://<AGIC-IP>/app2/index.html

# Verify Load Balancers
1. Load Balancer
2. Application Gateway
```

### Step-04-03: Clean-Up Apps
```t
# Delete Apps
kubectl delete -R -f kube-manifests
```



## Ingress Annotation Reference
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/

## Other References
- https://github.com/kubernetes/ingress-nginx
- https://github.com/kubernetes/ingress-nginx/blob/master/charts/ingress-nginx/values.yaml
- https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/cloud/deploy.yaml
- https://kubernetes.github.io/ingress-nginx/deploy/#azure
- https://helm.sh/docs/intro/install/
- https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#ingress-v1-networking-k8s-io
- [Kubernetes Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#ingress-v1-networking-k8s-io)
- [Ingress Path Types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types)

## Important Note
```
Ingress Admission Webhooks
With nginx-ingress-controller version 0.25+, the nginx ingress controller pod exposes an endpoint that will integrate with the validatingwebhookconfiguration Kubernetes feature to prevent bad ingress from being added to the cluster. This feature is enabled by default since 0.31.0.
```
