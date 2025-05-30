https://azureglobalblackbelts.com/2024/03/05/workload-identity-kv-csi.html


****Workload Identity - Key Vault CSI Driver****
---
**Cluster Creation**

First, lets create the AKS cluster with the OIDC Issuer, Workload Identity and the Key Vault CSI Driver add-ons enabled.

NOTE: Both the OIDC Issuer and Workload Identity add-ons are needed if you want to use workload identities, however the Key Vault CSI driver can also work with service principals or even the managed identity assigned to the cluster itself. Using Workload Identity, however, is the most versatile and secure approach.


```
  RG=WorkloadIdentityKVCSIRG
  LOC=eastus
  CLUSTER_NAME=wikvcsilab
  KEY_VAULT_NAME=key-test-7786

  # Create the resource group
az group create -g $RG -l $LOC

# Create the cluster with the OIDC Issuer and Workload Identity enabled
az aks create -g $RG -n $CLUSTER_NAME \
--node-count 1 \
--enable-oidc-issuer \
--enable-workload-identity \
--enable-addons azure-keyvault-secrets-provider \
--generate-ssh-keys

# Get the cluster credentials
az aks get-credentials -g $RG -n $CLUSTER_NAME
```

**Set up the identity**

In order to federate a managed identity with a Kubernetes Service Account we need to get the AKS OIDC Issure URL, create the Managed Identity and Service Account and then create the federation.

```
# Get the OIDC Issuer URL
export AKS_OIDC_ISSUER="$(az aks show -n $CLUSTER_NAME -g $RG --query "oidcIssuerProfile.issuerUrl" -otsv)"

# Get the Tenant ID for later
export IDENTITY_TENANT=$(az account show -o tsv --query tenantId)

# Create the managed identity
az identity create --name kvcsi-demo-identity --resource-group $RG --location $LOC

# Get identity client ID
export USER_ASSIGNED_CLIENT_ID=$(az identity show --resource-group $RG --name kvcsi-demo-identity --query 'clientId' -o tsv)

# Create a service account to federate with the managed identity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  labels:
    azure.workload.identity/use: "true"
  name: kvcsi-demo-sa
  namespace: default
EOF

# Federate the identity
az identity federated-credential create \
--name kvcsi-demo-federated-id \
--identity-name kvcsi-demo-identity \
--resource-group $RG \
--issuer ${AKS_OIDC_ISSUER} \
--subject system:serviceaccount:default:kvcsi-demo-sa

```

**Create the Key Vault and Secret**

```
# Create a key vault
az keyvault create --name $KEY_VAULT_NAME --resource-group $RG --location $LOC --enable-rbac-authorization false

# Create a secret
az keyvault secret set --vault-name $KEY_VAULT_NAME --name "TestSecret" --value "Hello from Key Vault"

# Grant access to the secret for the managed identity
az keyvault set-policy --name $KEY_VAULT_NAME -g $RG --secret-permissions get --spn "${USER_ASSIGNED_CLIENT_ID}"

#####################################################################
# OPTIONAL
# We'll get the version ID for the secret but this is not mandatory
#####################################################################

# Get the version ID
az keyvault secret show --vault-name $KEY_VAULT_NAME --name "TestSecret" -o tsv --query id
https://wi-demo-keyvault.vault.azure.net/secrets/Secret/ded8e5e3b3e040e9bfa5c47d0e28848a

# The version ID is the last part of the resource id above
# We'll use this later
VERSION_ID=ded8e5e3b3e040e9bfa5c47d0e28848a

```
The SecretProviderClass is what is used to create the binding between the Kubernetes cluster and the Azure Key Vault. It allows you to define the identity accessing the Key Vault, the target Key Vault name and the details about the keys and secrets being accessed. It will also let you link Key Vault Secrets to Kubernetes Secrets, but we’ll see more on that later.


```
# Create a secret provider instance attached to the vault and secret
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvcsi-wi # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEY_VAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: TestSecret             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: "${VERSION_ID}"               # [OPTIONAL] object versions, default to latest if empty
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
EOF

```

Now deploy a pod that gets the value using the service account identity.


```
# Create a pod to mount the secret
cat <<EOF | kubectl apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: secrets-store-inline-wi
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "kvcsi-demo-sa"
  containers:
    - name: ubuntu
      image: ubuntu:20.04
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
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
          secretProviderClass: "azure-kvcsi-wi"
EOF

# Check the secret loaded
kubectl exec -it secrets-store-inline-wi -- cat /mnt/secrets-store/TestSecret

```

**Sync the same secret to a Kubernetes Secret**

As mentioned above, you do have the option to synchronize Azure Key Vault secrets to Kubernetes secrets, which can be very useful. Just keep in mind that this driver uses the Container Storage Interface (CSI) so you still need to ensure a pod mounts a volume associated to the SecretProviderClass to initiate the Kubernetes secret creation.

Lets add a new SecretProviderClass that includes the secret synchronization configuration.

```
# Create a secret provider instance attached to the vault and secret
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvcsi-wi-sync # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "${USER_ASSIGNED_CLIENT_ID}" # Setting this to use workload identity
    keyvaultName: ${KEY_VAULT_NAME}       # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: TestSecret             # Set to the name of your secret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: "${VERSION_ID}"               # [OPTIONAL] object versions, default to latest if empty
    tenantId: "${IDENTITY_TENANT}"        # The tenant ID of the key vault
  secretObjects:                              # [OPTIONAL] SecretObjects defines the desired state of synced Kubernetes secret objects
  - data:
    - key: secretvalue                           # data field to populate
      objectName: TestSecret                        # name of the mounted content to sync; this could be the object name or the object alias
    secretName: foosecret                     # name of the Kubernetes secret object
    type: Opaque
EOF
```

Now, create a pod that loads the secret both as a volume and an environment variable from our synchronized Kubernetes secret.

```
# Create a pod to mount the secret
cat <<EOF | kubectl apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: secrets-store-inline-wi-sync
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: "kvcsi-demo-sa"
  containers:
    - name: ubuntu
      image: ubuntu:20.04
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
      env:
      - name: SECRET_DATA
        valueFrom:
          secretKeyRef:
            name: foosecret
            key: secretvalue
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-kvcsi-wi-sync"
EOF

```

Finally, test to make sure the secret is properly loaded to the volume and environment variable.

```
# Check that the secret was properly mounted as a volume
kubectl exec -it secrets-store-inline-wi-sync -- cat /mnt/secrets-store/TestSecret

# Check that the Kubernetes Secret was created
kubectl get secret foosecret -o jsonpath='{.data.secretvalue}'|base64 --decode

# Check that the secret was properly mounted from the kubernetes secret as an enviornment variable
kubectl exec -it secrets-store-inline-wi-sync -- /bin/bash -c 'echo $SECRET_DATA'

```






