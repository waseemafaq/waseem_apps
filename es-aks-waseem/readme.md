# Add the Elasticsearch Helm Repository
helm repo add elastic https://helm.elastic.co
helm repo update

# Install the Elastic Operator
helm install elastic-operator elastic/eck-operator --namespace elastic-system --create-namespace --version 2.11.0
# helm install elastic-operator elastic/eck-operator --namespace elastic-system --create-namespace --version 2.4.0

# Create a Namespace for Elasticsearch
kubectl create namespace elasticsearch

# Add Taints to the NodePool
export resourcegroup="waseem-afaq-poc"
export clustername="aks-dr-eastus"
export nodepoolname="worker"


az aks nodepool update \
    --resource-group $resourcegroup \
    --cluster-name $clustername \
    --name $nodepoolname \
    --node-taints "elasticsearch=true:NoSchedule"


# Add Labels to the NodePool
az aks nodepool update \
    --resource-group $resourcegroup \
    --cluster-name $clustername \
    --name $nodepoolname \
    --labels "elasticsearch=esdata"

# Apply Storage Class
kubectl apply -f storageclass.yaml

# Apply elastic search yaml
kubectl apply -f elasticsearch.yaml
kubectl apply -f kibana.yaml
