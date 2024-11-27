# Run the following command to get the node pools for your AKS cluster:
az aks nodepool list --resource-group waseem-afaq-poc --cluster-name aks-primary-eastus --output table

# Set Vairaibles
```t
$resourcegroup="waseem-afaq-poc"
$clustername="aks-primary-eastus"
$nodepoolname="wrkwas"
```

# Add Taints to the Node Pool
az aks nodepool update `
    --resource-group $resourcegroup `
    --cluster-name $clustername `
    --name $nodepoolname `
    --node-taints nodepool=waseemapp:NoSchedule

# Add Labels to the Node Pool
az aks nodepool update `
    --resource-group $resourcegroup `
    --cluster-name $clustername `
    --name $nodepoolname `
    --labels agentpool=wrkwas
