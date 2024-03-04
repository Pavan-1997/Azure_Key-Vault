# Azure_Key-Vault

1. Create Azure Resource Group

```
az group create --name keyvault-demo --location eastus
```

2. Create an AKS cluster with Azure Key Vault provider for Secrets Store CSI Driver support

```
az aks create --name keyvault-demo-cluster -g keyvault-demo --node-count 1 --enable-addons azure-keyvault-secrets-provider --enable-oidc-issuer --enable-workload-identity
```

3. Connect to the K8s cluster

```
az account set --subscription cfb43fdc-af53-41d8-9f37-626207b22277
```
```
az aks get-credentials --resource-group keyvault-demo --name keyvault-demo-cluster --overwrite-existing
```
```
kubectl config current-context
```
![CONTEXT](https://github.com/Pavan-1997/Azure_Key-Vault/assets/32020205/98d1dfae-20a0-4f5a-8ed4-39b45b4c9d44)


4. Verify that each node in your cluster's node pool has a Secrets Store CSI Driver pod and a Secrets Store Provider Azure pod running

```
kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)' -o wide
```
![CSI](https://github.com/Pavan-1997/Azure_Key-Vault/assets/32020205/1dcace50-dd03-40a8-9589-89bed610eec9)


5. Create a key vault with Azure role-based access control (Azure RBAC).

```
az keyvault create -n aks-demo-pavan -g keyvault-demo -l eastus --enable-rbac-authorization
```

6. Do the Role assignment

    Go to IAM -> Click on Add role assignment -> Select the Key Vault Administrator -> Next -> Select members -> Yourself -> Review + assign

    Create a Secret 


7. Connect your Azure ID to the Azure Key Vault Secrets Store CSI Driver

    Configure workload identity

```
export SUBSCRIPTION_ID=cfb43fdc-af53-41d8-9f37-626207b22277
export RESOURCE_GROUP=keyvault-demo
export UAMI=azurekeyvaultsecretsprovider-keyvault-demo-cluster
export KEYVAULT_NAME=aks-demo-pavan
export CLUSTER_NAME=keyvault-demo-cluster

az account set --subscription $SUBSCRIPTION_ID
```


8. Create a managed identity

```
az identity create --name $UAMI --resource-group $RESOURCE_GROUP

export USER_ASSIGNED_CLIENT_ID="$(az identity show -g $RESOURCE_GROUP --name $UAMI --query 'clientId' -o tsv)"
export IDENTITY_TENANT=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query identity.tenantId -o tsv)
```


9. Create a role assignment that grants the workload ID access the key vault

```
export KEYVAULT_SCOPE=$(az keyvault show --name $KEYVAULT_NAME --query id -o tsv)

az role assignment create --role "Key Vault Administrator" --assignee $USER_ASSIGNED_CLIENT_ID --scope $KEYVAULT_SCOPE
```

IF NOT

Go to Resource Group -> keyvault-demo -> Select Managed identity -> Managed identity - User-assigned -> Select - azurekeyvaultsecretsprovider-keyvault-demo-cluster
/subscriptions/cfb43fdc-af53-41d8-9f37-626207b22277/resourceGroups/MC_keyvault -> Select -> Click on Review + assign

10. Get the AKS cluster OIDC Issuer URL

export AKS_OIDC_ISSUER="$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)"
echo $AKS_OIDC_ISSUER


11. Create the service account for the pod

```
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_NAMESPACE="default" 

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
EOF


12. Setup Federation 

export FEDERATED_IDENTITY_NAME="aksfederatedidentity" 

az identity federated-credential create --name $FEDERATED_IDENTITY_NAME --identity-name $UAMI --resource-group $RESOURCE_GROUP --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}


13. Create the Secret Provider Class

cat <<EOF | kubectl apply -f -
# This is a SecretProviderClass example using workload identity to access your key vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-wi # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEYVAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: secret-pavan             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: ""               # [OPTIONAL] object versions, default to latest if empty
        - |
          objectName: key-pavan                # Set to the name of your key
          objectType: key
          objectVersion: ""
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
EOF


14. Create a sample pod to mount the secrets

cat <<EOF | kubectl apply -f -
# This is a sample pod definition for using SecretProviderClass and workload identity to access your key vault
kind: Pod
apiVersion: v1
metadata:
  name: busybox-secrets-store-inline-wi
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "workload-identity-sa"
  containers:
    - name: busybox
      image: registry.k8s.io/e2e-test-images/busybox:1.29-4
      command:
        - "/bin/sleep"
        - "10000"
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvname-wi"
EOF
