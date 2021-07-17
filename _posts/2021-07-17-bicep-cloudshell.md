---
title: "Deploy CloudShell to multiple subscriptions in an Enterprise Scale Scenario"
date: 2021-07-17T11:15:30+02:00
categories:
  - InfrastructureAsCode
tags:
  - bicep
  - azure
  - IaC
classes: wide
---

In this post I'm describing how you leverage Bicep modules to deploy azure resoruces to multiple subscriptions using a management group scope deployment. I'm showing an example on how to deploy a cloud shell storage account.

All files can be found on my GitHub [IaC repository](https://github.com/Egoorbis/iac/tree/main/bicep/cloudshell).

## Use case
Bicep offers several deployment scopes that can be targeted when deploying templates.
In an Enterprise Scale Szenario usually several core components are deployed centrally, while applications and workloads are deployed directly trough development teams. 
In our scenario we did have a large amount of subscriptions provisioned, so to ease up deployment we decided to deploy core components to management group level and specifc the subscription during deployment via parameters or variables. 

## Deploy to management group scope
In order to be able to achieve our goal, we need to create a ```main.bicep``` file with ```targetscope = 'managementgroup'``` and make use of the modules feature of Bicep. As well I prefer to use a parameters file. I have prepared the following files, that are also available in my public [git repository]()


```yaml
--main
  --main.bicep
--modules
  --rg.bicep
  --storageaccount.bicep
  --storageaccount.fileshare.bicep
--parameters
  --envParameters.json
```

### Parameters File
To load all required parameters I have created a dedicated file. I declared two parameter objects, that I can easily extend with additional parameters and use the same param file for multiple scenarios in a dedicated subscription, e. g. my prod environemnt.
As example I'm declaing envParameters such as ```subId, tags, environment, unit, location``` and then I declare parameters per service I want to deploy, in this example ```cloudshell```. For a storage accounts to be recognized as a Cloud Shell Storage, it must be tagged with ```ms-resource-usage: azure-cloud-shell```. As you can see this tag is specified under the ```cloudshell``` service, as it must only be used for this purpose.

Addtionally I create a second object, ```naming``` with the resource abbreviations which I use to concatenate the naming convention.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "envParameters": {
            "value": {
                "subId": "7420011a-d14c-4990-a610-2373dadd1309",
                "tags": {
                    "createdBy": "metr",
                    "environment": "prd",
                    "managedBy": "bicep"
                },
                "environment": "prd",
                "unit": "metr",
                "location": "westeurope",
                "service": {
                    "cloudshell": {
                        "sku": {
                            "Name": "Standard_LRS"
                        },
                        "kind": "StorageV2",
                        "tags": {
                            "ms-resource-usage": "azure-cloud-shell"
                        },
                        "properties": {
                            "minimumTlsVersion": "TLS1_2",
                            "allowBlobPublicAccess": false,
                            "supportsHttpsTrafficOnly": true,
                            "accessTier": "Hot"
                        }
                    }
                }
            }
        },
        "naming":{
            "value": {
                "resourceGroup": "rg",
                "storageAccount": "st"
            }
        }
    }
}
```

### main.bicep
Now let's take a look at the main.bicep file. As you can see at the very beginning, both objects, ```envParameters``` and ```naming```, are loaded and used to create most of the magic. I won't go into much detail, as I think this is more or less self-explanatory.

As descirbed above, the Cloud Shell has a specifc tag requirement. We use the union array function of Bicep in order to combine multiple tag sources together into a single array with all our tags.


```powershell
/*
Deploy a cloud shell account to management scope
*/
targetScope = 'managementGroup'

// Load parameters objects from parameters file
param envParameters object
param naming object

// Add creationDate parameter 
param creationDate object = {
  'CreationDate': utcNow('dd/MM/yyyy')
}

// Create resource group and storage account name based on envParameters and naming specified in parameters file
var rgName = '${envParameters.environment}-${envParameters.unit}-${naming.resourceGroup}-cloudshell'
var stName = '${envParameters.environment}${envParameters.unit}${naming.storageAccount}cloudshell'
var fsName = 'cloudshell'

// Deploy resource group to subscription scope
module rg_deploy '../modules/rg.bicep' = {
  name: 'deployment-${rgName}'
  scope: subscription(envParameters.subId)
  params: {
    name: rgName
    tags: union(creationDate, envParameters.tags)
    location: envParameters.location
  }
}

// Deploy storage account to resource group
module cs_deploy '../modules/storageaccount.bicep' = {
  name: 'deployment-${stName}'
  dependsOn: [
    rg_deploy
  ]
  scope: resourceGroup(envParameters.subId, rgName)
  params: {
    name: stName
    tags: union(creationDate, envParameters.tags, envParameters.service.cloudshell.tags)
    location: envParameters.location
    envParameters: envParameters.service.cloudshell
  }
}

module csfs_deploy '../modules/storageaccount.fileshare.bicep' = {
  name: 'deployment-${stName}-fileshare'
  dependsOn: [
    cs_deploy
  ]
  scope: resourceGroup(envParameters.subId, rgName)
  params: {
    stName: stName
    fsName: fsName
  }
}

output storageAccountName string = cs_deploy.outputs.storageAccountName
```

### Deployment
I will create an additional post with a Azure DevOps pipeline setup that has been used. 
To deploy the template manually use the following syntax:

```bash 
az deployment mg create -f main/main.bicep -p parameters/envparameters.json -m %managementGroupID% -l westeurope
```


:page_facing_up: *Sources:* 
* Create Bicep parameter file - [https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/parameter-files](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/parameter-files)
* Management group deployments with Bicep files - [https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-to-management-group?tabs=azure-cli](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-to-management-group?tabs=azure-cli)
* Persist files in Azure Cloud Shell - [https://docs.microsoft.com/en-us/azure/cloud-shell/persisting-shell-storage#restrict-resource-creation-with-an-azure-resource-policy](https://docs.microsoft.com/en-us/azure/cloud-shell/persisting-shell-storage#restrict-resource-creation-with-an-azure-resource-policy)
