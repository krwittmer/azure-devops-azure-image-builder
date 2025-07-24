
# 🛠️ Developer Guide: Using Azure DevOps with Azure Image Builder

This guide walks you through setting up and using a fully functional Azure DevOps pipeline to deploy and run an Azure Image Builder ARM template.

---

## 📦 Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1: Setup Service Connection](#step-1-setup-service-connection)
- [Step 2: Prepare Your Azure Resources](#step-2-prepare-your-azure-resources)
- [Step 3: Create the ARM Template](#step-3-create-the-arm-template)
- [Step 4: Create the Azure DevOps Pipeline](#step-4-create-the-azure-devops-pipeline)
- [Step 5: Trigger the Pipeline](#step-5-trigger-the-pipeline)
- [🔧 Substitutions Required](#-substitutions-required)

---

## ✅ Overview

We will:
- Define an ARM template to create a custom VM image using Azure Image Builder.
- Deploy and run this template via a multi-stage Azure DevOps pipeline.
- Use service principal-based authentication via an Azure DevOps service connection.

---

## ✅ Prerequisites

- Azure subscription with rights to create Managed Identities and Images.
- Azure DevOps project with permissions to create pipelines.
- Service Principal (created via portal or CLI).
- Installed Azure CLI or access to Azure Cloud Shell.

---

## 🚀 Step 1: Setup Service Connection

1. **Create a Service Principal** (if you don’t already have one):

```bash
az ad sp create-for-rbac --name DevOpsImageBuilder --role Contributor --scopes /subscriptions/<subscriptionId>
```

2. **In Azure DevOps**:
   - Navigate to `Project Settings` > `Service connections`
   - Click `New service connection` > `Azure Resource Manager`
   - Choose **Service principal (manual)** and input:
     - Subscription ID
     - Subscription Name
     - Tenant ID
     - Service Principal ID (AppId)
     - Service Principal Key (Password)
   - Name it `YourServiceConnectionName`

---

## 🏗️ Step 2: Prepare Your Azure Resources

- Resource Group for Image Builder Template: `aib-resource-group`
- Resource Group for Staging: `<stagingRg>`
- Managed Identity Resource Group: `<identityRg>`
- Image Output Resource Group: `<imageOutputRg>`

Ensure all required resource groups and managed identities exist with proper roles.

---

## 📄 Step 3: Create the ARM Template

Save this file as `imageTemplate.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.VirtualMachineImages/imageTemplates",
      "apiVersion": "2023-07-01",
      "name": "demoImageTemplate",
      "location": "eastus",
      "identity": {
        "type": "UserAssigned",
        "userAssignedIdentities": {
          "/subscriptions/<subscriptionId>/resourceGroups/<identityRg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<identityName>": {}
        }
      },
      "properties": {
        "buildTimeoutInMinutes": 90,
        "source": {
          "type": "PlatformImage",
          "publisher": "Canonical",
          "offer": "UbuntuServer",
          "sku": "18.04-LTS",
          "version": "latest"
        },
        "customize": [
          {
            "type": "Shell",
            "name": "UpdatePackages",
            "inline": [
              "sudo apt-get update",
              "sudo apt-get upgrade -y"
            ]
          }
        ],
        "distribute": [
          {
            "type": "ManagedImage",
            "imageId": "/subscriptions/<subscriptionId>/resourceGroups/<imageOutputRg>/providers/Microsoft.Compute/images/demoImage",
            "location": "eastus",
            "runOutputName": "demoManagedImage"
          }
        ],
        "stagingResourceGroup": "/subscriptions/<subscriptionId>/resourceGroups/<stagingRg>",
        "vmProfile": {
          "vmSize": "Standard_D2_v2"
        },
        "optimize": {
          "vmBoot": {
            "state": "Enabled"
          }
        }
      }
    }
  ]
}
```

---

## ⚙️ Step 4: Create the Azure DevOps Pipeline

Save this file as `azure-pipelines.yml`:

```yaml
trigger:
- main

variables:
  resourceGroup: 'aib-resource-group'
  imageTemplateFile: 'imageTemplate.json'
  imageTemplateName: 'demoImageTemplate'
  location: 'eastus'
  apiVersion: '2023-07-01'

stages:
- stage: DeployTemplate
  jobs:
  - job: DeployAIBTemplate
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: AzureCLI@2
      displayName: 'Deploy Azure Image Builder Template'
      inputs:
        azureSubscription: 'YourServiceConnectionName'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          az deployment group create             --resource-group $(resourceGroup)             --template-file $(imageTemplateFile)

- stage: RunAIBTemplate
  dependsOn: DeployTemplate
  jobs:
  - job: RunAIB
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: AzureCLI@2
      displayName: 'Run Azure Image Builder'
      inputs:
        azureSubscription: 'YourServiceConnectionName'
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          az resource invoke-action             --resource-group $(resourceGroup)             --resource-type Microsoft.VirtualMachineImages/imageTemplates             --name $(imageTemplateName)             --action Run             --api-version $(apiVersion)
```

---

## ▶️ Step 5: Trigger the Pipeline

1. Push `azure-pipelines.yml` and `imageTemplate.json` to your `main` branch.
2. Navigate to Pipelines in Azure DevOps.
3. Run the pipeline manually or wait for automatic trigger on `main`.

---

## 🔧 Substitutions Required

In the ARM template and pipeline YAML:

| Placeholder            | Replace with                                  |
|------------------------|-----------------------------------------------|
| `<subscriptionId>`     | Your Azure subscription ID                    |
| `<identityRg>`         | Resource group containing the managed identity |
| `<imageOutputRg>`      | Output image destination resource group       |
| `<stagingRg>`          | Staging resource group                        |
| `<identityName>`       | Name of the user-assigned managed identity    |
| `YourServiceConnectionName` | The name of the Azure DevOps service connection |

---

## ✅ Summary

You now have:
- A reusable ARM template for Azure Image Builder.
- An Azure DevOps pipeline to deploy and run that template.
- A service connection to securely interact with Azure resources.

