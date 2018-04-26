## Part 1: Using Azure to deploy OpenShift Origin
### 1.0: Open cloud shell and log in with your Azure account you created in the Prework
In case you forgot, here's a helpful link: (https://shell.azure.com)

### 1.1: Create new Resource Groups with `az group create`
You will create 2 resource groups. One resource group will be used to host your
Azure key vault and the other will be used to host your OpenShift cluster resources.
```bash
> az group create --name <KEYVAULT_RESOURCE_GROUP_NAME> --location westus2

> az group create --name <RESOURCE_GROUP_NAME> --location westus2
```
**Pro Tip:** You can be even lazier with what you type in the CLI. Examples include:
  1. The `--name` argument can be shortened to just `-n`
  1. The `--resource-group` argument can be shortened to just `-g`
  1. The `--location` argument can be shortened to just `-l`


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
    --name <YOUR_SERVICE_PRINCIPAL_NAME> \
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
  "password": "<YOUR_PASSWORD_HERE>",
  "tenant": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
}
```
**Notes:**
  * Save the appId field from the output for later. You will need it. If you
  forget to save it and now see this line because you did a Ctrl+F for 'appId' from a
  later step, you can get the app id from the Azure Portal.
    * On the left, click on Azure Active Directory
    * Click on App Registrations in the blade that pops up, and find the appId
    corresponding to the Service Principal you created earlier
  * Azure CLI commands are sorted hierarchically. In this case, `az ad sp create-for-rbac`
  expands to "Azure -> Active Directory -> Service Principal -> Create for Role-
  Based Access".
  * Your SubscriptionId can be found with `az account list` (from above).
  * Your Resource Group name can be found with `az group list`.
  * The final `scopes` argument limits the scope of the Service Principal to the
  current Resource Group. This is a generally accepted best practice.

### 1.5: Create your Azure Key Vault
Create a key vault to store the SSH keys for the cluster with the az keyvault
create command. The key vault name must be globally unique, be between 3 and 24
characters, and contain only letters and numbers. Use the Key Vault resource
group you provisioned earlier for this step.
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
```bash
> git clone https://github.com/Microsoft/openshift-origin.git
```

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
    ```bash
    "masterInstanceCount": {
        "type": "int",
        "defaultValue": 3,
        "allowedValues": [1,5],   # Change from [3,5] to [1,5]
        "metadata": {
            "description": "Number of OpenShift masters."
        }
    },
    "infraInstanceCount": {
        "type": "int",
        "defaultValue": 2,
        "allowedValues": [1,3],   # Change from [2,3] to [1,3]
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

    #### Changes required for `azuredeploy.parameters.local.json`:
    ```bash
    {
      "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
      "contentVersion": "1.0.0.0",
      "parameters": {
          "_artifactsLocation": {
              "value": "https://raw.githubusercontent.com/Microsoft/openshift-origin/master/"
          },
          "masterVmSize": {
              "value": "Standard_DS2_v2"
          },
          "infraVmSize": {
              "value": "Standard_DS2_v2"
          },
          "nodeVmSize": {
              "value": "Standard_DS2_v2"
          },
          "storageKind": {
              "value": "managed"
          },
          "openshiftClusterPrefix": {
              "value": "changeme"     # Your cluster name here (this must have a length of at most 20 characters)
          },
          "masterInstanceCount": {
              "value": 1              # Change from 3 to 1
          },
          "infraInstanceCount": {
              "value": 1              # Change from 2 to 1
          },
          "nodeInstanceCount": {
              "value": 2              # Change from 1 to 2
          },
          "dataDiskSize": {
              "value": 128
          },
          "adminUsername": {
              "value": "changeme"     # Your admin name here
          },
          "openshiftPassword": {
              "value": "changeme"     # Your password here
          },
          "enableMetrics": {
              "value": "false"
          },
          "enableLogging": {
              "value": "false"
          },
          "enableCockpit": {
              "value": "false"
          },
          "sshPublicKey": {
              "value": "changeme"     # Add your SSH public key here
          },
          "keyVaultResourceGroup": {
              "value": "changeme"     # Add your Key Vault resource group nanme here
          },
          "keyVaultName": {
              "value": "changeme"     # Add the name of your Key Vault here
          },
          "keyVaultSecret": {
              "value": "changeme"     # Add the name of your  Key Vault Secret here
          },
          "enableAzure": {
              "value": "true"         # Change this from false to true
          },
          "aadClientId": {
              "value": "changeme"     # Add the AppId you saved from earlier here
          },
          "aadClientSecret": {
              "value": "changeme"     # Add the password for your AAD Service Principal here
          },
          "defaultSubDomainType": {
              "value": "nipio"
          },
          "defaultSubDomain": {
              "value": "changeme"     # If you selected nipio above, this will be ignored
          }
      }
    }

    ```

### 1.8: Deploy OpenShift!
Use the Azure Resource Manager group deployment command. Note that the resource
group you use here will be the OpenShift resource group you created earlier - NOT
the Key Vault resource group (remember - you created 2 resource groups!)
  ```bash
  > az group deployment create \
    --resource-group <YOUR_RESOURCE_GROUP_HERE> \
    --template-file azuredeploy.local.json \
    --parameters @azuredeploy.parameters.local.json
  ```
You are now finished with this part of the lab. Deployment will take anywhere between
25-40 minutes, so you may now continue onto the next part of this lab.The URL of
the OpenShift console and the DNS name of the OpenShift master prints to the
terminal when the deployment finishes. It will look something like this:

  ```json
  {
    "OpenShift Console Uri": "http://openshiftlb.cloudapp.azure.com:8443/console",
    "OpenShift Master SSH": "ssh clusteradmin@myopenshiftmaster.cloudapp.azure.com -p 2200"
  }
  ```
If you lose this output, you can always retrieve the URL with the following:
 ```
    > az group deployment list \
      --resource-group <YOUR_RESOURCE_GROUP_HERE>
      --query [].properties.outputs \
      --output=json
  ```
