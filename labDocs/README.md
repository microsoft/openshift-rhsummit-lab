# Deploying OpenShift Origin on Microsoft Azure

## [Part 0: Pre-work, Getting an Azure Subscription](Part0.md) 


## Part 1: Using Azure to deploy OpenShift Origin
### 1.0: Open cloud shell and log in with your Azure account you created in the Prework
In case you forgot, here's a helpful link: (https://shell.azure.com)

### 1.1: Create a new Resource Group with `azure group create`
This resource group will be used to host your Azure key vault. You will deploy a
separate resource group for your OpenShift cluster resources.
```bash
> azure group create --name <KEYVAULT_RESOURCE_GROUP_NAME> --location westus2
```
**Pro Tip:** You can be even lazier with what you type by typing `az` instead of `azure`.
The rest of this guide will use `az` instead of `azure` for Azure CLI commands.

### 1.2: Get your subscription's SubscriptionId with `az account list`
```bash
> az account list
```

### 1.3: Generate SSH Keys with `ssh-keygen`
```bash
> ssh-keygen
```
Ensure that no passphrase is created for the key, and note down where the key
pair is saved (probably ~/.ssh/).

### 1.4: Create your Azure Service Principal
```bash
> az ad sp create-for-rbac \
    --name <YOUR_NAME_HERE> \
    --password <YOUR_PASSWORD_HERE> \
    --role contributor \
    --scopes /subscriptions/<YOUR_SUBSCRIPTION_ID>/resourceGroups/<YOUR_RESOURCE_GROUP_NAME> 
```
Your output will look something like this:
```json
{
  "appId": "11111111-abcd-1234-efgh-111111111111",            
  "displayName": "<YOUR_NAME_HERE>",
  "name": "http://<YOUR_NAME_HERE>",
  "password": <YOUR_PASSWORD_HERE>,
  "tenant": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
}
```
**Notes:**  
  * Azure CLI commands are sorted hierarchically. In this case, `az ad sp create-for-rbac`
  expands to "Azure -> Active Directory -> Service Principal -> Create for Role-
  Based Access". 
  * Your SubscriptionId can be found with `az account list` (from above).
  * Your Resource Group name can be found with `az group list`.
  * The final `scopes` argument limits the scope of the Service Principal to the
  current Resource Group. This is a generally accepted best practice.

### 1.5: Create your Azure Key Vault
Create a key vault to store the SSH keys for the cluster with the az keyvault
create command. The key vault name must be globally unique.
```bash
> az keyvault create \
    --resource-group <KEYVAULT_RESOURCE_GROUP_NAME> \
    --name <KEYVAULT_NAME> \
    --location westus2 \
    --enabled-for-template-deployment true
```
### 1.6: Store your SSH private key in the Azure Key Vault
The OpenShift deployment uses the SSH key you created to secure access to the OpenShift master. To enable the deployment to securely retrieve the SSH key, store the key in Key Vault.
```bash
> az keyvault secret set \
    --vault-name <KEYVAULT_NAME> \
    -- name <SECRET_NAME> \
    --file <PATH_TO_SSH_PRIVATE_KEY> 
```
**Note:**
  * The path to your SSH private key is probably ~/.ssh/id_rsa

### 1.7: Download the OpenShift deployment repository
You can use one of two ways to deploy OpenShift Origin on Azure:

  * You can manually deploy all the necessary Azure infrastructure components, and
then follow the OpenShift Origin documentation.
  * You can also use an existing Resource Manager template that simplifies the
deployment of the OpenShift Origin cluster.

Today, we will be using the template to automate the deployment of all the
necessary Azure infrastructure components and run the setup scripts required.

1. Go to (https://aka.ms/openshift) and clone
the repo to your local machine (in today's lab, this can all be done within Azure
Cloud Shell).
1. Enter the cloned repo directory and create copies of `azuredeploy.json` and
`azuredeploy.parameters.json`
    ```bash
    > cd openshift-origin
    > cp azuredeploy.json azuredeploy.local.json
    > cp azuredeploy.parameters.json azuredeploy.parameters.local.json
    ```
1. Use your preferred text editor to make adjustments to both files. We will be
updating `azuredeploy.local.json` to allow for fewer Master nodes. This is due
to a 10 core constraint in the Azure Pass. The bulk of our editing work will be
in `azuredeploy.parameters.local.json`. There are a number of fields tagged with
a "changeme" string that will need to be edited.
    #### Changes required for `azuredeploy.local.json`:
    ```json
    "masterInstanceCount": {
        "type": "int",
        "defaultValue": 3,
        "allowedValues": [1,5],
        "metadata": {
            "description": "Number of OpenShift masters."
        }
    },
    "infraInstanceCount": {
        "type": "int",
        "defaultValue": 2,
        "allowedValues": [1,3],
        "metadata": {
            "description": "Number of OpenShift infra nodes."                                                            }
    },
    "nodeInstanceCount": {
        "type": "int",
        "defaultValue": 2,
        "minValue": 1,
        "allowedValues": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18,  20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30],
        "metadata": {                                                       
            "description": "Number of OpenShift nodes"
        }
    }
    ```
