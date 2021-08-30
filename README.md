# AKS with App Gateway ingress controller and certificates from Key Vault as `kubernetes.io/tls` secrets

This work is a generalization of the work done by [@AAkindele](https://github.com/AAkindele) in the Gist: https://gist.github.com/AAkindele/d341cca3b56f51a49f1585f47e6ad2f3. Here, I am tokenizing the scripts that [@AAkindele](https://github.com/AAkindele) wrote and leveraging the steps and descriptions from that Gist.

----

These instructions will show you how to create an [AKS](https://docs.microsoft.com/en-us/azure/aks/) Kubernetes cluster that leverages [Azure Application Gateway](https://docs.microsoft.com/en-us/azure/application-gateway/overview) as the cluster's [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). It will also create an [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/) to store the TLS certificates that will be used.

At deployment:

- The Kubernetes cluster will pull the certificate from the Key Vault using the [CSI driver](https://docs.microsoft.com/en-us/azure/aks/csi-secrets-store-driver) and the certificates will be stored as a `kubernetes.io/tls` secret.
- The Application Gateway will use the certificates from Kubernetes `kubernetes.io/tls` secret as the TLS certificate
- The service will use the Application gateway as the ingress controller with an HTTPS endpoint 

----

## Variables set-up

To ensure the scripts will run properly, first open a PowerShell window and run the following (replacing the variable values with your own).\
If you do not already have the [Azure CLI installed](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli), you will need this to connect to Azure via `az login` and to run the scripts

``` PowerShell
# Set names the Azure resource values
$RESOURCE_GROUP="mm-agic-demo"
$AKS_CLUSTER_NAME="mm-demo-aks"
$APPGW_NAME="mm-demo-agw"
$KEYVAULT_NAME="mm-kv-demo"
$LOCATION="eastus"

# Set names and values for the identity
$POD_IDENTITY_NAME="mm-demo-pod-identity"
$POD_IDENTITY_NAMESPACE="mm-demo"
$SECRETS_PROVIDER_CLASS_NAME="mwm-demo-csi-provider"
$AAD_TENANT_ID="" # GUID for your Azure AD tenant

# Set host name and certificate information 
$HOST_NAME="temptest.com"
$CERTIFICATE_PATH=""  # path to certificate file
$CERTIFICATE_KEY=""   # path to certificate key
```

## Setup preview extensions

```bash
# add az cli preview extensions
az extension add --name aks-preview
az extension update --name aks-preview
az feature register --name EnablePodIdentityPreview --namespace Microsoft.ContainerService
az feature register --name AutoUpgradePreview --namespace Microsoft.ContainerService
```

## Create the resource group and AKS cluster

``` PowerShell
az group create -n $RESOURCE_GROUP -l $LOCATION
az aks create --name $AKS_CLUSTER_NAME --resource-group $RESOURCE_GROUP --network-plugin azure --enable-managed-identity --enable-addons ingress-appgw --appgw-name $APPGW_NAME --appgw-subnet-cidr "10.2.0.0/16"

# enable pod identity. this can be done on cluster create as well
az aks update --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --enable-pod-identity

# get cluster creds
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME
```

## Create the Azure Key Vault and upload certificate

If you have certs in a key vault already, you can use that vault instead of creating a new one.

*I used secrets key vault here here instead of certificate resource because certificate I'm using includes a ca bundle. The bundle is not saved to k8s secrets when using certificate directly.*



``` PowerShell
az keyvault create --name $KEYVAULT_NAME --resource-group $RESOURCE_GROUP -l $LOCATION

# set secrets in key vault
# an alternative is to upload to a key vault certificate instead of using individual secrets.
az keyvault secret set -n tls-crt --vault-name $KEYVAULT_NAME --file $CERTIFICATE_PATH
az keyvault secret set -n tls-key --vault-name $KEYVAULT_NAME --file $CERTIFICATE_KEY
```

## Set up AAD Pod Identity

``` PowerShell
$MANAGED_RG_NAME = (az aks show --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME -o tsv --query "nodeResourceGroup")
$USER_ASSIGNED_MI_NAME = (az identity list --resource-group $MANAGED_RG_NAME -o tsv --query "[?contains(@.name 'ingress')].name")
$IDENTITY_CLIENT_ID="$(az identity show --resource-group $MANAGED_RG_NAME --name $USER_ASSIGNED_MI_NAME --query clientId -otsv)"
$IDENTITY_RESOURCE_ID="$(az identity show --resource-group $MANAGED_RG_NAME --name $USER_ASSIGNED_MI_NAME --query id -otsv)"
$IDENTITY_PRINCIPAL_ID="$(az identity show --resource-group $MANAGED_RG_NAME --name $USER_ASSIGNED_MI_NAME --query principalId -otsv)"

$NODES_RESOURCE_ID=$(az group show --name $MANAGED_RG_NAME -o tsv --query "id")
az role assignment create --role "Virtual Machine Contributor" --assignee "$IDENTITY_PRINCIPAL_ID" --scope $NODES_RESOURCE_ID
az keyvault set-policy --name $KEYVAULT_NAME --object-id $IDENTITY_PRINCIPAL_ID --secret-permissions get list

az aks pod-identity add --resource-group $RESOURCE_GROUP --cluster-name $AKS_CLUSTER_NAME --namespace $POD_IDENTITY_NAMESPACE  --name $POD_IDENTITY_NAME --identity-resource-id $IDENTITY_RESOURCE_ID
```

## Setup Key Vault CSI provider

**IMPORTANT:** The `secrets-store-csi-driver.syncSecret.enabled` option is required for creating the Kubernetes secrets.

``` bash
helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts
helm repo update
helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --namespace kube-system --set secrets-store-csi-driver.syncSecret.enabled=true
```

## Deploy sample app

### Perform token replacement on YAML files
``` PowerShell
$DEPLOYMENT_YAML = (Get-Content deployment.yaml) -replace "##POD_IDENTITY_NAMESPACE##", $POD_IDENTITY_NAMESPACE -replace "##POD_IDENTITY_NAME##", $POD_IDENTITY_NAME -replace "##SECRETS_PROVIDER_CLASS_NAME##", $SECRETS_PROVIDER_CLASS_NAME

$INGRESS_YAML = (Get-Content ingress.yaml) -replace "##POD_IDENTITY_NAMESPACE##", $POD_IDENTITY_NAMESPACE -replace "##HOST_NAME##", $HOST_NAME

$SECRET_PROVIDER_YAML= (Get-Content secret-provider.yaml) -replace "##POD_IDENTITY_NAMESPACE##", $POD_IDENTITY_NAMESPACE -replace "##KEYVAULT_NAME##", $KEYVAULT_NAME -replace "##SECRETS_PROVIDER_CLASS_NAME##", $SECRETS_PROVIDER_CLASS_NAME -replace "##AAD_TENANT_ID##", $AAD_TENANT_ID

$SERVICE_YAML= (Get-Content service.yaml) -replace "##POD_IDENTITY_NAMESPACE##", $POD_IDENTITY_NAMESPACE
```

### Deploy

``` PowerShell
# create secret provider
$SECRET_PROVIDER_YAML | kubectl apply -f -

# deploy sample app
$DEPLOYMENT_YAML | kubectl apply -f -
$SERVICE_YAML | kubectl apply -f -
$INGRESS_YAML | kubectl apply -f -
```

## Verify

You can verify that the certificate was pulled from Key Vault into the Kubernetes secret with:
``` Powershell
kubectl get secrets -n $POD_IDENTITY_NAMESPACE
```

And that the ingress controller is leveraging that secret with:

``` PowerShell
kubectl describe ingress aspnetapp -n $POD_IDENTITY_NAMESPACE
```
