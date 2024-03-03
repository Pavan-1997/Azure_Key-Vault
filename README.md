# Azure_Key-Vault

1. Create Azure Resource Group

```
az group create --name keyvault-demo --location eastus
```

2. Create an AKS cluster with Azure Key Vault provider for Secrets Store CSI Driver support
az aks create --name keyvault-demo-cluster -g keyvault-demo --node-count 1 --enable-addons azure-keyvault-secrets-provider --enable-oidc-issuer --enable-workload-identity


3. Connect to the K8s cluster

az account set --subscription cfb43fdc-af53-41d8-9f37-626207b22277

az aks get-credentials --resource-group keyvault-demo --name keyvault-demo-cluster --overwrite-existing

kubectl config current-context


4. Verify that each node in your cluster's node pool has a Secrets Store CSI Driver pod and a Secrets Store Provider Azure pod running

kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)' -o wide


5. Create a key vault with Azure role-based access control (Azure RBAC).

az keyvault create -n aks-demo-pavan -g keyvault-demo -l eastus --enable-rbac-authorization


6. Do the Role assignment

Go to IAM -> Click on Add role assignment -> Select the Key Vault Administrator -> Next -> Select members -> Yourself -> Review + assign


Create a Secret 

7. Connect your Azure ID to the Azure Key Vault Secrets Store CSI Driver

Configure workload identity

export SUBSCRIPTION_ID=cfb43fdc-af53-41d8-9f37-626207b22277
export RESOURCE_GROUP=keyvault-demo
export UAMI=azurekeyvaultsecretsprovider-keyvault-demo-cluster
export KEYVAULT_NAME=aks-demo-pavan
export CLUSTER_NAME=keyvault-demo-cluster

az account set --subscription $SUBSCRIPTION_ID


8. Create a managed identity

az identity create --name $UAMI --resource-group $RESOURCE_GROUP

export USER_ASSIGNED_CLIENT_ID="$(az identity show -g $RESOURCE_GROUP --name $UAMI --query 'clientId' -o tsv)"
export IDENTITY_TENANT=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query identity.tenantId -o tsv)


9. Create a role assignment that grants the workload ID access the key vault

export KEYVAULT_SCOPE=$(az keyvault show --name $KEYVAULT_NAME --query id -o tsv)

az role assignment create --role "Key Vault Administrator" --assignee $USER_ASSIGNED_CLIENT_ID --scope $KEYVAULT_SCOPE

IF NOT

Go to Resource Group -> keyvault-demo -> Select Managed identity -> Managed identity - User-assigned -> Select - azurekeyvaultsecretsprovider-keyvault-demo-cluster
/subscriptions/cfb43fdc-af53-41d8-9f37-626207b22277/resourceGroups/MC_keyvault -> Select -> Click on Review + assign
