
# Azure Image Builder: Macro Substitution in ARM JSON Templates

## 📌 Problem
ARM templates written in JSON do not support native macro substitution like Bicep or YAML. Strings such as:

```json
"stagingResourceGroup": "/subscriptions/<subscriptionId>/resourceGroups/<stagingRg>"
```

...must be replaced externally before template deployment.

---

## ✅ Recommended Solutions

### 1. Azure DevOps Pipeline Variable Injection

Use PowerShell or Bash in your DevOps pipeline to inject values dynamically.

**Example (PowerShell in Azure DevOps):**

```yaml
- task: AzureCLI@2
  displayName: 'Inject JSON parameters dynamically using PowerShell'
  inputs:
    azureSubscription: 'YourServiceConnectionName'
    scriptType: 'ps'
    scriptLocation: 'inlineScript'
    inlineScript: |
      $json = Get-Content 'imageTemplate.json' -Raw
      $json = $json -replace '<subscriptionId>', $env:SUBSCRIPTION_ID
      $json = $json -replace '<stagingRg>', 'aib-staging-rg'
      Set-Content 'imageTemplate-processed.json' $json
```

You then deploy the processed file:

```bash
az deployment group create   --resource-group aib-resource-group   --template-file imageTemplate-processed.json
```

---

### 2. Use ARM Template Parameters

Structure your ARM JSON to use ARM template parameters and a `.parameters.json` file:

**`imageTemplate.json` (partial):**
```json
"stagingResourceGroup": "[parameters('stagingRg')]"
```

**`imageTemplate.parameters.json`:**
```json
{
  "parameters": {
    "subscriptionId": { "value": "xxxx-..." },
    "stagingRg": { "value": "aib-staging-rg" }
  }
}
```

Then deploy:

```bash
az deployment group create   --resource-group aib-resource-group   --template-file imageTemplate.json   --parameters @imageTemplate.parameters.json
```

---

### 3. Use Bicep (Recommended for New Projects)

Bicep supports native variable substitution:

```bicep
param subscriptionId string
param stagingRg string

resource imageTemplate 'Microsoft.VirtualMachineImages/imageTemplates@2023-07-01' = {
  name: 'demoImageTemplate'
  location: 'eastus'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '/subscriptions/${subscriptionId}/resourceGroups/identity-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/demo-identity': {}
    }
  }
  properties: {
    stagingResourceGroup: '/subscriptions/${subscriptionId}/resourceGroups/${stagingRg}'
  }
}
```

Then deploy with:

```bash
az deployment group create   --resource-group aib-resource-group   --template-file main.bicep   --parameters subscriptionId=$SUB_ID stagingRg=$STAGING_RG
```

---

### 4. Use Template Engines (Jinja2, etc.)

You can also use tools like **Jinja2**, Python scripts, or Mustache templates, though these add complexity.

---

## ✅ Recommendation Summary

| Approach                        | Pros                                      | Cons                                     |
|---------------------------------|-------------------------------------------|------------------------------------------|
| ARM Parameters File             | Native support, clean separation          | Slightly verbose                         |
| PowerShell/Bash String Replace | Quick for simple use cases                | Can be brittle with large JSON files     |
| Bicep                          | Best for modern, maintainable templates   | Requires Bicep CLI and familiarity       |
| Templating Engines              | Very flexible                             | Adds external dependencies and tooling   |

---

## 📘 Best Practice

For DevOps pipelines:
- Use **ARM parameters** for stable, reusable templates.
- Use PowerShell/Bash only when values are dynamic at runtime.
- Use **Bicep** for modern ARM template authoring.

Let me know if you need a reusable script or helper for one of these workflows!
