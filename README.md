# Demo for AKS Workload Identity
[[Deploy and configure workload identity on an Azure Kubernetes Service (AKS) cluster](https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster)]

Azure Kubernetes Service (AKS) is a managed Kubernetes service that lets you quickly deploy and manage Kubernetes clusters. This article shows you how to:
- Deploy an AKS cluster using the Azure CLI with the OpenID Connect issuer and a Microsoft Entra Workload ID.
- Create a Microsoft Entra Workload ID and Kubernetes service account.
- Configure the managed identity for token federation.
- Deploy the workload and verify authentication with the workload identity.
- Optionally grant a pod in the cluster access to secrets in an Azure key vault.

## Prerequisites
- If you don't have an Azure subscription, create an Azure free account before you begin.
- This article requires version 2.47.0 or later of the Azure CLI. If using Azure Cloud Shell, the latest version is already installed.
- Make sure that the identity that you're using to create your cluster has the appropriate minimum permissions. For more information about access and identity for AKS, see Access and identity options for Azure Kubernetes Service (AKS).
- If you have multiple Azure subscriptions, select the appropriate subscription ID in which the resources should be billed using the az account set command.

## Set Active Subscription

```Console
az login --tenant 16b3c013-d300-468d-ac64-7eda0820b6d3

az account set --subscription ce9b673a-0158-4c19-9adf-d76a7f63f498`
```

## Export environmental variables

```console
export RESOURCE_GROUP="rg-demo-identity-aks-simple"
export LOCATION="eastus" \
export CLUSTER_NAME="oidc-aks" \
export SERVICE_ACCOUNT_NAMESPACE="default" \
export SERVICE_ACCOUNT_NAME="workload-identity-sa" \
export SUBSCRIPTION="$(az account show --query id --output tsv)" \
export USER_ASSIGNED_IDENTITY_NAME="aksIdentity" \
export FEDERATED_IDENTITY_CREDENTIAL_NAME="myaksFedIdentity" \`
# Include these variables to access key vault secrets from a pod in the cluster.
export KEYVAULT_NAME="keyvault-workload-id" \
export KEYVAULT_SECRET_NAME="aks-secret" \
```

## Create and configure Azure Resources
### Create Resource Group

```Console
az group create --name "${RESOURCE_GROUP}" --location "${LOCATION}"
```

### Create AKS Cluster

```Console
az aks create --resource-group "${RESOURCE_GROUP}" --name "${CLUSTER_NAME}" --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys
```

### Update existing AKS Cluster

```console
az aks update --resource-group "${RESOURCE_GROUP}" --name "${CLUSTER_NAME}" --enable-oidc-issuer --enable-workload-identity
```
### Retrieve the OIDC issuer URL

```Console
export AKS_OIDC_ISSUER="$(az aks show --name "${CLUSTER_NAME}" --resource-group "${RESOURCE_GROUP}" --query "oidcIssuerProfile.issuerUrl" --output tsv)"
```

### Create a managed identity
```Console
az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}"
```

### Create a variable for the managed identity

```Console
export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' --output tsv)"
```

## Create a AKS Service Account

```Console
az aks get-credentials --name "${CLUSTER_NAME}" --resource-group "${RESOURCE_GROUP}"
```

### Export the $KUBECONFIG variable to the merge config file

[https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/](https://)


```Console
export KUBECONFIG=/mnt/c/Users/dawahby/.kube/config
```
### Create and apply yaml to create a k8s service account
```Console
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: "${USER_ASSIGNED_CLIENT_ID}"
  name: "${SERVICE_ACCOUNT_NAME}"
  namespace: "${SERVICE_ACCOUNT_NAMESPACE}"
EOF
```
## Create the federated identity credential

```Console
az identity federated-credential create --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --issuer "${AKS_OIDC_ISSUER}" --subject system:serviceaccount:"${SERVICE_ACCOUNT_NAMESPACE}":"${SERVICE_ACCOUNT_NAME}" --audience api://AzureADTokenExchange
```

Comment: paste the variable ${AKS_OIDC_ISSUER} add an \r at the end of the traling /. Paste the complete string creates the issuer right.

## Deploy de application
### Create Azure Key Vault
```Console
export KEYVAULT_RESOURCE_GROUP="rg-demo-identity-aks-simple"
export KEYVAULT_NAME="dwi-kv-aks-wli-050624"

az keyvault create --name "${KEYVAULT_NAME}" --resource-group "${KEYVAULT_RESOURCE_GROUP}" --location "${LOCATION}" --enable-purge-protection --enable-rbac-authorization
```
### Assign current user as Secrets Officer
This way the signed-in user can create the secret

```Console
export KEYVAULT_RESOURCE_ID=$(az keyvault show --resource-group "${KEYVAULT_RESOURCE_GROUP}" --name "${KEYVAULT_NAME}" --query id --output tsv)

az role assignment create --assignee "dawahby_microsoft.com#EXT#@fdpo.onmicrosoft.com" --role "Key Vault Secrets Officer" --scope "${KEYVAULT_RESOURCE_ID}"
``` 
With SPN does not work

```Console
az role assignment create --assignee-object-id "6df383c3-9cfa-4775-836d-f3e5c86a13f6" --assignee-principal-type "User" --role "Key Vault Secrets Officer" --scope "${KEYVAULT_RESOURCE_ID}"
```
**At the end the permissions were assigned manually thru Portal**

### Create secret in the KV

```Console
export KEYVAULT_SECRET_NAME="my-secret"

az keyvault secret set \
    --vault-name "${KEYVAULT_NAME}" \
    --name "${KEYVAULT_SECRET_NAME}" \
    --value "Hello\!"
```

### Assing the Key Vault Secrets User to the managed identity
```Console
export IDENTITY_PRINCIPAL_ID=$(az identity show --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --query principalId --output tsv)

az role assignment create --assignee-object-id "${IDENTITY_PRINCIPAL_ID}" --role "Key Vault Secrets User" --scope "${KEYVAULT_RESOURCE_ID}" --assignee-principal-type ServicePrincipal
```

### Create and enviroment variable for the KV URL

```Console
export KEYVAULT_URL="$(az keyvault show --resource-group ${KEYVAULT_RESOURCE_GROUP} --name ${KEYVAULT_NAME} --query properties.vaultUri --output tsv)"
```

### Deploy a pod that references the service account and KV URL
```Console
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sample-workload-identity-key-vault
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}
  containers:
    - image: ghcr.io/azure/azure-workload-identity/msal-go
      name: oidc
      env:
      - name: KEYVAULT_URL
        value: ${KEYVAULT_URL}
      - name: SECRET_NAME
        value: ${KEYVAULT_SECRET_NAME}
  nodeSelector:
    kubernetes.io/os: linux
EOF
```
### Check that the pod have the secret env in its configuration the secret from KV
```Console
kubectl describe pod sample-workload-identity-key-vault | grep "SECRET_NAME:"
```

### Check that the pod can read the secret
```Console
kubectl logs sample-workload-identity-key-vault
```

