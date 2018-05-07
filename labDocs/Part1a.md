## Part 1a: Troubleshooting or Restarting OpenShift Origin Deployment
If the deployment fails, the goal is to understand where and why and restart it. This section will cover most common cases and we may address individual issues during the lab.

### Bad template parameters
If clearly see that parameters are wrong based on the error message do the following:

1. Delete the existing resource group (OpenShiftRG1)
```bash
az group delete --name OpenShiftRG1 --no-wait
```
2. Create a new resource group (OpenShiftRG2)
```bash
az group create --name OpenShiftRG2 --location westus2
```
3. Grant the permissions of your service principal to the newly created group
```bash
az role assignment create \
    --role Contributor \
    --assignee-object-id `az ad sp list --display-name "OpenShiftSPName" --query '[].objectId' --output=tsv` \
    --resource-group OpenShiftRG2
```
3. Modify the parameters and save them

4. Check that prior group has been deleted (or rather that you have enough cores). Azure Pass subscription is limited to 10 cores total by default and if you have any VMs not yet deleted, you may hit core quota issues
```bash
az vm list-usage --location WestUS2
```
Check that you have at least 8 cores available. 

5. Resubmit the deployment adjusting the group name
```bash
az group deployment create \
  --template-file azuredeploy.local.json \
  --parameters @azuredeploy.parameters.local.json \
  --resource-group OpenShiftRG2
```

[Back to TOC](../README.md) | [Using OpenShift](Part2.md)