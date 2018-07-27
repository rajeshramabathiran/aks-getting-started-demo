# Getting started
This document outlines the steps involved to setup a managed container cluster (AKS) in Azure, using Azure CLI. The intent is to capture a set of steps that can be repeated consistently in order to setup a managed container cluster in Azure.

These steps vary from the vanialla steps available in Azure documentation in a few ways:
- Resource tagging
- Secure store for secrets

**Resource tagging**

All resources deployed following the steps below are tagged with an `APPNAME` tag. This makes it really easy to filter down the set of resources in Azure subscription specific to this app. 

> An additional benefit of tagging is the ability to view the billing info at a tag level. Since all resources are tagged with `APPNAME`, you can get the billing info at the app level easily instead of manaully curating that information.

**Secure store for secrets**

It is a good practice to store all crucial info in a secure store. Azure has a secure store called Key Vault. The secrets and other key information required to setup the cluster are stored in an Azure Key Vault.


## Pre-requisites

### Setting up Azure CLI
To install the Azure CLI, follow the steps documented in [Azure documentation][1].

Login to Azure
```bash
az login
```

List all Azure suscriptions
```bash
az account list --output table
```

Set the default Azure subscription for CLI use
```bash
AZURE_SUBSCRIPTION_ID="<azure-subscription-id>"
az account set --subscription $AZURE_SUBSCRIPTION_ID
```

Check if the default subscription is set correctly
```bash
az account show
```

### Create Azure Keyvault store

All resource groups created as part of creating the managed cluster in Azure will be prefixed with a configurable string.

```bash
RESOURCE_GROUP_PREFIX="aks-demo101-"
COMMON_RESOURCE_GROUP="common"
KEYVAULT_RESOURCE_GROUP="$RESOURCE_GROUP_PREFIX$COMMON_RESOURCE_GROUP"
LOCATION="West US"
DEMO_TAG="AKS101"
```

Create resource group
```bash
az group create --name $KEYVAULT_RESOURCE_GROUP --location $LOCATION --tags "AppName=$DEMO_TAG"
```

View the list of resource groups specific to the demo
```bash
az group list --tag "APPNAME=$DEMO_TAG" --output table
```

Create the keyvault resource
```bash
KEYVAULT_RESOURCE_POSTFIX="keystore"
KEYVAULT_RESOURCE_NAME="$RESOURCE_GROUP_PREFIX$KEYVAULT_RESOURCE_POSTFIX"
az keyvault create --name $KEYVAULT_RESOURCE_NAME --resource-group $KEYVAULT_RESOURCE_GROUP --location $LOCATION --tags "AppName=$DEMO_TAG"
```

List all resources specific to the demo
```bash
az resource list --tag "APPNAME=$DEMO_TAG" --output table   
```

### Create a key for AKS cluster

Generate SSH Key for demo cluster
```bash
SSH_KEY_COMMENT_PHRASE="azureuser@$DEMO_TAG"
SSH_KEY_FILE_NAME="/Users/rajeshramabathiran/.ssh/AKS101"
SSH_KEY_PASSPHRASE="$DEMO_TAG"

ssh-keygen \
    -t rsa \
    -b 4096 \
    -C $SSH_KEY_COMMENT_PHRASE \
    -f $SSH_KEY_FILE_NAME \
    -N $SSH_KEY_PASSPHRASE
```

Upload key to Key Vault
```bash
SSHKEY_LOOKUP="sshkey"
az keyvault secret set --name $SSHKEY_LOOKUP --vault-name $KEYVAULT_RESOURCE_NAME --file $SSH_KEY_FILE_NAME --tags "AppName=$DEMO_TAG"

SSHKEY_LOOKUP_PUB="sshkeypub"
az keyvault secret set --name $SSHKEY_LOOKUP_PUB --vault-name $KEYVAULT_RESOURCE_NAME --file "$SSH_KEY_FILE_NAME.pub" --tags "AppName=$DEMO_TAG"
```

Download public key locally, if needed
```bash
SSH_KEY_PUB_LOCAL_PATH="$SSH_KEY_FILE_NAME.pub"
az keyvault secret download --name $SSHKEY_LOOKUP_PUB --vault-name $KEYVAULT_RESOURCE_NAME --file $SSH_KEY_PUB_LOCAL_PATH
```

### Create AKS cluster

Create AKS resource group
```bash
CLUSTER_RESOURCE_GROUP="cluster"
AKS_RESOURCE_GROUP="$RESOURCE_GROUP_PREFIX$CLUSTER_RESOURCE_GROUP"
az group create --name $AKS_RESOURCE_GROUP --location $LOCATION --tags "AppName=$DEMO_TAG"
```

>    https://stedolan.github.io/jq/ - Command line JSON processor
>    `brew install jq`

>    Example usage: `myaccount=$(az account show)`

>    `environmentName=$(echo $myaccount | jq --raw-output '.environmentName')`

Create service principal for AKS cluster access
```bash
CLUSTER_SERVICE_PRINCIPAL=$(az ad sp create-for-rbac --name $DEMO_TAG --skip-assignment)
CLUSTER_SERVICE_PRINCIPAL_APP_ID=$(echo $CLUSTER_SERVICE_PRINCIPAL | jq --raw-output '.appId')
CLUSTER_SERVICE_PRINCIPAL_DISPLAY_NAME=$(echo $CLUSTER_SERVICE_PRINCIPAL | jq --raw-output '.displayName')
CLUSTER_SERVICE_PRINCIPAL_NAME=$(echo $CLUSTER_SERVICE_PRINCIPAL | jq --raw-output '.name')
CLUSTER_SERVICE_PRINCIPAL_TENANT=$(echo $CLUSTER_SERVICE_PRINCIPAL | jq --raw-output '.additionalProperties.appOwnerTenantId')
CLUSTER_SERVICE_PRINCIPAL_PASSWORD=$(echo $CLUSTER_SERVICE_PRINCIPAL | jq --raw-output '.password')
```

Upload service principal info to Key Vault
```bash
CLUSTER_SP_LOOKUP_APP_ID="CLUSTER-SERVICE-PRINCIPAL-APP-ID"
CLUSTER_SP_LOOKUP_DISPLAY_NAME="CLUSTER-SERVICE-PRINCIPAL-DISPLAY-NAME"
CLUSTER_SP_LOOKUP_NAME="CLUSTER-SERVICE-PRINCIPAL-NAME"
CLUSTER_SP_LOOKUP_PASSWORD="CLUSTER-SERVICE-PRINCIPAL-PASSWORD"
CLUSTER_SP_LOOKUP_TENANT="CLUSTER-SERVICE-PRINCIPAL-TENANT"
```

```bash
az keyvault secret set --name $CLUSTER_SP_LOOKUP_APP_ID --vault-name $KEYVAULT_RESOURCE_NAME --value $CLUSTER_SERVICE_PRINCIPAL_APP_ID --tags "AppName=$DEMO_TAG"
```
> Note: Parameter 'secret_name' must conform to the following pattern: '^[0-9a-zA-Z-]+$'.

```bash
az keyvault secret set --name $CLUSTER_SP_LOOKUP_DISPLAY_NAME --vault-name $KEYVAULT_RESOURCE_NAME --value $CLUSTER_SERVICE_PRINCIPAL_DISPLAY_NAME --tags "AppName=$DEMO_TAG"
```

```bash
az keyvault secret set --name $CLUSTER_SP_LOOKUP_NAME --vault-name $KEYVAULT_RESOURCE_NAME --value $CLUSTER_SERVICE_PRINCIPAL_NAME --tags "AppName=$DEMO_TAG"
```

```bash
az keyvault secret set --name $CLUSTER_SP_LOOKUP_PASSWORD --vault-name $KEYVAULT_RESOURCE_NAME --value $CLUSTER_SERVICE_PRINCIPAL_PASSWORD --tags "AppName=$DEMO_TAG"
```

```bash
az keyvault secret set --name $CLUSTER_SP_LOOKUP_TENANT --vault-name $KEYVAULT_RESOURCE_NAME --value $CLUSTER_SERVICE_PRINCIPAL_TENANT --tags "AppName=$DEMO_TAG"
```

Create AKS with existing service principal
```bash
AKS_CLUSTER_NAME=$DEMO_TAG
az aks create --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER_NAME --service-principal $CLUSTER_SERVICE_PRINCIPAL_APP_ID --client-secret $CLUSTER_SP_LOOKUP_PASSWORD --ssh-key-value $SSH_KEY_PUB_LOCAL_PATH --tags "AppName=$DEMO_TAG"
```

<!--References -->
[1]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest