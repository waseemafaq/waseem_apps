# Set the subsctiption
$subscription = "8c01c8db-a164-4c51-bdbd-f46bd0779d16"
az account set --subscription $subscription
Set-AzContext -SubscriptionId $subscription

# 1. Create Azure Container Registry (ACR)
# Variables
```t
$ACR_NAME="waseemacr"
$RESOURCE_GROUP="waseem-afaq-poc"
$LOCATION="eastus"
$clustername="aks-primary-eastus"
$nodepoolname="elasticnp"
$newsku="Standard_D2s_v3"
$oldpool="workload"
```

# Create ACR
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic --location $LOCATION

# Next, log in to your ACR to authenticate your session.

az acr login --name $ACR_NAME --expose-token

# Copy Docker Image from Docker Hub to ACR

# Import eck-operator image into ACR with the same name
az acr import --name $ACR_NAME --source docker.elastic.co/eck/eck-operator:2.4.0 --image eck/eck-operator:2.4.0

# Import elasticsearch image into ACR with the same name
az acr import --name $ACR_NAME --source docker.elastic.co/elasticsearch/elasticsearch:8.4.3 --image elasticsearch/elasticsearch:8.4.3

# Kibana image into ACR
az acr import --name $ACR_NAME --source docker.elastic.co/kibana/kibana:8.4.3 --image kibana/kibana:8.4.3

# List all repo in the acr
az acr repository list --name $ACR_NAME --output table


================================================================

# Add nodepool, taint and labels
# Set Vairaibles
```t
$RESOURCE_GROUP="waseem-afaq-poc"
$clustername="aks-primary-eastus"
$nodepoolname="elasticnp"
$newsku="Standard_D2s_v3"
$oldpool="workload"

```

# Create a new node pool with the desired SKU
```t
az aks nodepool add `
    --resource-group $RESOURCE_GROUP `
    --cluster-name $clustername `
    --name $nodepoolname `
    --node-count 2 `
    --node-vm-size $newsku `
    --mode User `
    --max-pods 30 `
    --no-wait

# Add Taints to the Node Pool
az aks nodepool update `
    --resource-group $RESOURCE_GROUP `
    --cluster-name $clustername `
    --name $nodepoolname `
    --node-taints elasticsearch=true:NoSchedule

# Add Labels to the Node Pool
az aks nodepool update `
    --resource-group $RESOURCE_GROUP `
    --cluster-name $clustername `
    --name $nodepoolname `
    --labels elasticsearch=esdata
```

=========================================================

# Attach ACR to the AKS cluster.
az aks update `
  --resource-group $RESOURCE_GROUP `
  --name $clustername `
  --attach-acr $ACR_NAME

# Apply storage class storageclass.yaml file
kubectl apply -f .\storageclass.yaml

# Helm install eck operator
helm install elastic-operator elastic/eck-operator `
  --namespace elastic-system `
  --create-namespace `
  --version 2.4.0 `
  --set image.repository=waseemacr.azurecr.io/eck/eck-operator,image.tag=2.4.0

# Deploy ES 
kubectl apply -f elasticsearch.yaml

