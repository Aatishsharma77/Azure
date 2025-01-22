***SETUP AND CONFIGURE OF KEYVAULT*** 
---

**Create Azure Resource Group**
```
az group create --name keyvault-demo --location eastus
```

**AKS Creation and Configuration**

*Create an AKS cluster with Azure Key Vault provider for Secrets Store CSI Driver support*

```
az aks create --name keyvault-demo-cluster -g keyvault-demo --node-count 1 --enable-addons azure-keyvault-secrets-provider --enable-oidc-issuer --enable-workload-identity
```
**Get the Kubernetes cluster credentials (Update kubeconfig)**

```
az aks get-credentials --resource-group keyvault-demo --name keyvault-demo-cluster
```

**Verify that each node in your cluster's node pool has a Secrets Store CSI Driver pod and a Secrets Store Provider Azure pod running**

```
kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver,secrets-store-provider-azure)' -o wide
```

**Keyvault creation and configuration**
*Create a key vault with Azure role-based access control (Azure RBAC).*
```
az keyvault create -n demo-keyvault-7766 -g keyvault-demo -l eastus --enable-rbac-authorization
```
```
NOTE: After create keyvault -> Go to Secret, Key -> You see authorization issue "You are unauthorized to view these contents." -> Go to IAM -> Grant access to this resource -> Add Role Assignment -> Key Vault Administrator -> select -> User, group, or service principal -> select member -> add member -> Review and Assign.

```

***Connect your Azure ID to the Azure Key Vault Secrets Store CSI Driver***
---

**Configure workload identity**

```

export SUBSCRIPTION_ID=45cc2250-5c20-4675-b8e8-e8813daff840
export RESOURCE_GROUP=keyvault-demo
# Managed Identity
export UAMI=azurekeyvaultsecretsprovider-keyvault-demo-cluster
export KEYVAULT_NAME=demo-keyvault-7766
export CLUSTER_NAME=keyvault-demo-cluster

az account set --subscription $SUBSCRIPTION_ID
```

**Create a managed identity**
```
az identity create --name $UAMI --resource-group $RESOURCE_GROUP

export USER_ASSIGNED_CLIENT_ID="$(az identity show -g $RESOURCE_GROUP --name $UAMI --query 'clientId' -o tsv)"
export IDENTITY_TENANT=$(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query identity.tenantId -o tsv)
```

**Create a role assignment that grants the workload ID access the key vault**
```
export KEYVAULT_SCOPE=$(az keyvault show --name $KEYVAULT_NAME --query id -o tsv)

az role assignment create --role "Key Vault Administrator" --assignee $USER_ASSIGNED_CLIENT_ID --scope $KEYVAULT_SCOPE
```
```
NOTE: Above command not working somethime -> go keyvault -> demo-keyvault-7766 -> IAM -> Grant access to this resource -> click add role assignment -> select role "Key Vault Administrator" -> assign access to "Managed identity" -> select key vault.

```
![alt text](image-1.png)


**Get the AKS cluster OIDC Issuer URL**
```
export AKS_OIDC_ISSUER="$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)"
echo $AKS_OIDC_ISSUER
```

**Create the service account for the pod**

```
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_NAMESPACE="default" 
```
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
EOF
```
**Setup Federation**
```
export FEDERATED_IDENTITY_NAME="aksfederatedidentity" 

az identity federated-credential create --name $FEDERATED_IDENTITY_NAME --identity-name $UAMI --resource-group $RESOURCE_GROUP --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}
```

**Create the Secret Provider Class**

```
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
          objectName: secret1             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: ""               # [OPTIONAL] object versions, default to latest if empty
        - |
          objectName: key1                # Set to the name of your key
          objectType: key
          objectVersion: ""
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
EOF
```

***Create a sample pod to mount the secrets***
---

```
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
```

```
kubectl exec busybox-secrets-store-inline-wi -- sh 
cd /mnt/secrets-store
cat secret1
cat key1 
```