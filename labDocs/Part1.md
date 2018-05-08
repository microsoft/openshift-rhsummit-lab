## Part 1: Using Azure to deploy OpenShift Origin
In today's lab, we will be deploying OpenShift Origin clusters. OpenShift Origin
is an open-source upstream community-supported project of OpenShift.
OpenShift Container Platform is an enterprise-ready commercial version supported by Red Hat for which customers purchase the necessary entitlements.

For the purposes of this lab we will be using OpenShift Origin.

### Goals of this section:
* Get familiar with basics of deployment using Azure Resource Manager templates
* Generate artifacts reusable across deployments (ssh key pair, Azure Keyvault; Azure Service Principal)
* Create parameters (configuration) file for deployment
* Start the deployment
* Understand the process behind the deployment
* Get familiar with basic deployment troubleshooting

### 1.0: Open cloud shell and log in with your Azure account you created in the Prework
You can use one in the Azure Portal (http://portal.azure.com) or a standalone Cloud Shell window (https://shell.azure.com) or if you have installed Azure CLI, you can use the terminal on that machine. Do log in with the credentials you have used to redeem your Azure Pass.

### 1.1: Create new Resource Groups with `az group create`
Azure resouce group is a concept that allows one to group resources together based on their expected lifetime.

For this reason you will create 2 resource groups. One resource group will be used to host your
Azure key vault and associated private keys and the other will be used to host your OpenShift cluster resources.

If you need to restart (or repeat later) the deployment process, it is easier to delete the OpenShift cluster resource group since they KeyVault resource group can be reused. It is a best practice to group resources into the resource groups based on their expected lifetime.


```bash
az group create --name KeyvaultRG --location westus2

az group create --name OpenShiftRG1 --location westus2
```
**Pro Tip:** You can also use short parameter names with what you type in the CLI. Examples include:
  1. The `--name` argument can be shortened to just `-n`
  1. The `--resource-group` argument can be shortened to just `-g`
  1. The `--location` argument can be shortened to just `-l`

### 1.2: Get your subscription's SubscriptionId with `az account list`
```bash
az account list --output=table
```
**Note:**
If you want to change the default output type for az CLI, use `az configure`

#### 1.2.1 Verify and set your default subscription to Azure Pass subscription
If you do have multiple subscriptions, ensure you have selected the desired one with

```bash
az account set --subscription YourDesiredSubscriptionID
```

Use

```bash
az account show
```

to verify.


### 1.3: Generate SSH Keys with `ssh-keygen`
```bash
ssh-keygen
```
Ensure that no passphrase is created for the key, and note down where the key
pair is saved (probably ~/.ssh/).

Do not overwrite your existing keys and move the newly created key pair into `~/.ssh/` if necessary.

Passwordless key pair is required for the successful deployment or the Ansible playbooks will block waiting for the password to be entered. We do recommend a new key pair for this deployment.

If you want to use an existing key pair, the following commands may be useful if you need to remove the password from it or generate a public key from an existing private key:

1. To remove password from an existing key:
```bash
ssh-keygen -p -f ~/.ssh/id_rsa_openshift
```

2. To generate public key from the private key file (if you don't have one already):
```bash
ssh-keygen -y -f ~/.ssh/id_rsa_openshift > ~/.ssh/id_rsa_openshift.pub
```

### 1.4: Create your Azure Service Principal
Service Principal (SP) is a special form of identity which allows applications to make changes to your resources. It is given as a parameter to the deployment and would allow later adding or removing resources on behalf of the OpenShift cluster (for example, adding or removing data disks as persisted volumes).

We will create SP and assign its permissions in two steps.

First, create the SP

```bash
az ad sp create-for-rbac --name OpenShiftSPName --skip-assignment --output=jsonc
```

**Note**
We are explicitly skipping the role assignment and we use auto-generated password. If you prefer, you can specify explicit password instead.

Your output will look something like this:
```json
{
  "appId": "e0d1cd93-93d3-40af-8bf7-63edac400153",
  "displayName": "OpenShiftSPName",
  "name": "http://OpenShiftSPName",
  "password": "5d40e723-78c8-4726-ba04-dccd0a592909",
  "tenant": "1e07e96f-6950-459d-a380-743c8d449894"
}
```

**Notes:**
  * Save the output of the above command in a safe place. These are your "secrets", safeguard them as you would any password information.
  * If you forget to save the password, it cannot be recovered, you can only reset it. You can do so via Azure Portal or Azure CLI.
    * On the left, click on Azure Active Directory
    * Click on App Registrations in the blade that pops up, and find the ap by Display Name or App ID corresponding to the Service Principal you created earlier
  * If you want to start from scratch, you can also delete the SP and create a new one
  * You only need to create the SP once and then you can reuse it for multiple deployments.

Second step is to grant appropriate permissions to the newly created service principal on your subscription at the desired scope. In this case we would be granting the contributor permission to the resource group.

To do so use the following command:

```
az role assignment create \
    --role Contributor \
    --assignee-object-id `az ad sp list --display-name "OpenShiftSPName" --query '[].objectId' --output=tsv` \
    --resource-group OpenShiftRG1
```

We will go over this command during the lab to discuss its simpler version and the best practices on roles and scopes.

**Note** if you need to restart the deployment into a new resource group, you could use the above command to assign an existing service principal permissions to the new group. Just update the name of the resource group. Forgetting to grant permission to the service principal is one of the common reasons of deployment failures.


### 1.5: Create your Azure Key Vault
Create a key vault to store the SSH keys for the cluster with the az keyvault
create command. The key vault name must be globally unique, be between 3 and 24
characters, and contain only letters and numbers. Use the Key Vault resource
group you provisioned earlier for this step.

```bash
az keyvault create \
    --resource-group KeyvaultRG \
    --name KeyvaultOpenShift1001 \
    --location westus2 \
    --enabled-for-template-deployment true
```

### 1.6: Store your SSH private key in the Azure Key Vault
The OpenShift deployment uses the SSH key you created to secure access to the OpenShift master. To enable the deployment to securely retrieve the SSH key, store the key in Key Vault.
```bash
az keyvault secret set \
    --vault-name KeyvaultOpenShift1001 \
    --name OpenShiftPK \
    --file ~/.ssh/id_rsa_openshift
```
**Note:**
  * Adjust the name of the Keyvault and the path to the desired SSH private key


**This completes a set of preparation steps that could be reused on this subscription for multiple deployments**

### 1.7: Download the OpenShift deployment template repository
Next step is to get the Azure Resource Manager deployment template, set its parameters and start the actual deployment.

Using an existing Azure Resource Manager template simplifies the deployment of the OpenShift Origin cluster significantly and allows modifications to suit your configuration requirements.

Today, we will be using the template to automate the deployment of all the
necessary Azure infrastructure components and run the setup scripts required.

Go to (https://aka.ms/openshift) and clone
the repo to your local machine (in today's lab, this can all be done within Azure
Cloud Shell).
```bash
cd ~
git clone https://github.com/Microsoft/openshift-origin.git
```

Enter the cloned repo directory and create copies of `azuredeploy.json` and
`azuredeploy.parameters.json`

    ```bash
    cd openshift-origin
    cp azuredeploy.json azuredeploy.local.json
    cp azuredeploy.parameters.json azuredeploy.parameters.local.json
    ```

1. Use your preferred text editor to make adjustments to `azuredeploy.parameters.local.json`.
We will be updating this file to allow for fewer master and infra nodes. This is due to a
10 core constraint in the Azure Pass. In addition, there are a number of fields tagged with
a "changeme" string that will need to be edited. These set various parameters for
the OpenShift Deployment.

#### Changes required for `azuredeploy.local.json`:

```bash
    "masterInstanceCount": {
        "type": "int",
        "defaultValue": 3,
        "allowedValues": [1,3,5],   # Change from [3,5] to [1,3,5]
        "metadata": {
            "description": "Number of OpenShift masters."
        }
    },
    "infraInstanceCount": {
        "type": "int",
        "defaultValue": 2,
        "allowedValues": [1,2,3],   # Change from [2,3] to [1,2,3]
        "metadata": {
            "description": "Number of OpenShift infra nodes."
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
            "value": "Standard_DS2_v2"  # Change from Standard_DS3_v2 to Standard_DS2_v2
        },
        "infraVmSize": {
            "value": "Standard_DS2_v2"  # Change from Standard_DS3_v2 to Standard_DS2_v2
        },
        "nodeVmSize": {
            "value": "Standard_DS2_v2"  # Change from Standard_DS3_v2 to Standard_DS2_v2
        },
        "storageKind": {
            "value": "managed"          # Change to managed
        },
        "openshiftClusterPrefix": {
            "value": "openshift1001"    # Your cluster name here (length at most 20 characters)
        },
        "masterInstanceCount": {
            "value": 1                  # Change from 3 to 1
        },
        "infraInstanceCount": {
            "value": 1                  # Change from 2 to 1
        },
        "nodeInstanceCount": {
            "value": 2                  # Change from 1 to 2
        },

        ...

        "adminUsername": {
            "value": "adminuser"        # Your admin name here
        },
        "openshiftPassword": {
            "value": "DoNotCheckMeIn!"  # Your password here
        },

        ...

        "sshPublicKey": {
            "value": "ssh-rsa AAAA...OJa3pY+B+92...-v1cpk"    # SSH public key matching what you put in the keyvault
        },
        "keyVaultResourceGroup": {
            "value": "KeyvaultRG"                             # Add your Key Vault resource group name here
        },
        "keyVaultName": {
            "value": "KeyvaultOpenShift1001"                  # Add the name of your Key Vault here
        },
        "keyVaultSecret": {
            "value": "OpenShiftPK"                            # Add the name of your Key Vault Secret here
        },
        "enableAzure": {
            "value": "true"                                   # Check this is set to true
        },
        "aadClientId": {
            "value": "e0d1cd93-93d3-40af-8bf7-63edac400153"   # Add the AppId you saved earlier
        },
        "aadClientSecret": {
            "value": "5d40e723-78c8-4726-ba04-dccd0a592909"   # Add the password for your Service Principal
        },
        "defaultSubDomainType": {
            "value": "nipio"
        },
        "defaultSubDomain": {
            "value": "changeme"                               # If you selected nipio above, this will be ignored
        }
    }
}
```

### 1.8: Deploy OpenShift
The final step is to submit the deployment to Azure.

You will be using the second resource group you created (OpenShiftRG1), not the one with the keyvault.

To start the deployment, use this command
```bash
az group deployment create \
  --template-file azuredeploy.local.json \
  --parameters @azuredeploy.parameters.local.json \
  --resource-group OpenShiftRG1
```
You are now finished with this part of the lab. Deployment will take anywhere between
25-45 minutes, so you may now continue onto the next part of this lab. The URL of
the OpenShift console and the DNS name of the OpenShift master prints to the
terminal when the deployment finishes. It will look something like this:

```json
{
"OpenShift Console Uri": "http://openshift1001.cloudapp.azure.com:8443/console",
"OpenShift Master SSH": "ssh adminuser@myopenshiftmaster.cloudapp.azure.com -p 2200"
}
```
If you lose this output (or the session disconnects), you can always retrieve the URL with the following:
```bash
az group deployment list \
--resource-group OpenShiftRG1
--query '[].properties.outputs' \
--output=json
```

Or via Azure portal on by clicking on the deployments associated with this resource group.

[Back to TOC](../README.md) | [Troubleshooting](Part1a.md) | [Using OpenShift](Part2.md)
