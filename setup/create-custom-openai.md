# Create Custom OpenAI

This environment provisions a Custom Azure OpenAI resource within a dedicated resource group, designed to support intelligent schema conversion and modernization workflows during Oracle to PostgreSQL migration. The OpenAI model is integrated into the migration tooling to enhance decision-making, automate transformation logic, and assist with documentation generation.

In this repository, the Azure OpenAI resource is primarily used to:

- Provide language model support for schema analysis and conversion recommendations
- Assist in generating migration scripts, transformation mappings, and validation queries
- Enhance documentation clarity and automation throughout the migration lifecycle
- Enable intelligent interaction and troubleshooting during complex migration scenarios

⚠️ **Note:** Replace placeholder values like `<your-subscription-id>` with your own before running the commands.

---

### Step 1: Set your subscription

```powershell
# Set your subscription ID
$subscriptionId="<your-subscription-id>"

# Set the Azure subscription context
az account set --subscription $subscriptionId
```

💡 Note:
The $subscriptionId variable stores your Azure subscription.
By using az account set, all future CLI commands will run under this subscription context.

### Step 2: Define general variables

```powershell
# Variables
$location="westus3"
$resourceGroup="rg-openai01"
$vnet="vnet-openai01"
$backendSubnet="subnet-vnet-openai01-backend"
$backendSubnetNSG="nsg-subnet-vnet-openai01-backend"
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$jumpboxResourceGroup="rg-jumpbox"
$jumpboxVNet="vnet-jumpbox"
$serviceName="AzOpenAIModern01"
$customSubdomain = "AzOpenAIModern01CSD"
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for the Custom OpenAI

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

### Step 4: Create the Azure Custom OpenAI virtual network and subnet

```powershell
# Create a virtual network with a subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.7.0.0/16 `
  --subnet-name $backendSubnet `
  --subnet-prefix 10.7.0.0/24 `
  --location $location
```

### Step 5: Create network security group (NSG) and attach it to the subnet

```powershell
# Create a network security group
az network nsg create `
  --resource-group $resourceGroup `
  --name $backendSubnetNSG `
  --location $location

# Attach network security group to virtual network subnet
az network vnet subnet update `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --name $backendSubnet `
  --network-security-group $backendSubnetNSG
```

### Step 6: Peer spoke vnets to hub and jumpbox

```powershell
# Peer openai01 to hub
az network vnet peering create `
  --name openai01-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

# Peer hub to openai01
az network vnet peering create `
  --name hub-to-openai01 `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  

# Peer openai01 to jumpbox
az network vnet peering create `
  --name openai01-to-jumpbox `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$jumpboxResourceGroup/providers/Microsoft.Network/virtualNetworks/$jumpboxVNet" `
  --allow-vnet-access

# Peer jumpbox to openai01
az network vnet peering create `
  --name jumpbox-to-openai01 `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  
```

### Step 7: Link the shared Private DNS Zone

Before creating the Custom OpenAI server, link the shared private DNS zone (privatelink.openai.azure.com) hosted in rg-hub to the VNet where your Custom OpenAI will reside.

> 💡 Why?
- Private endpoints rely on Azure Private DNS Zones to resolve the *.openai.azure.com names to private IPs.
- This ensures your Custom OpenAI is reachable only within your VNets, not over the public internet.

```powershell
# Link your vNet (vnet-openai01) to DNS zone (privatelink.openai.azure.com)
# So, that resources inside the vNet can resolve names like *.openai.azure.com
az network private-dns link vnet create `
  --resource-group $hubResourceGroup `
  --zone-name privatelink.openai.azure.com `
  --name link-to-vnet-openai01-openai `
  --virtual-network "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --registration-enabled false
```

> ⚠️ Note:
The link must be created in the same resource group as the DNS zone (rg-hub), even if your Custom OpenAI is deployed in another group (rg-openai01).

### Step 8: Create Custom OpenAI

```powershell
# Create Custom OpenAI
az cognitiveservices account create `
  --name $serviceName `
  --resource-group $resourceGroup `
  --kind OpenAI `
  --sku S0 `
  --location $location

# Disable Public Network Access 
az resource update `
  --name $serviceName `
  --resource-group $resourceGroup `
  --resource-type "Microsoft.CognitiveServices/accounts" `
  --set properties.publicNetworkAccess="Disabled"

# Enable Public Network Access if needed
<#
az resource update `
  --name $serviceName `
  --resource-group $resourceGroup `
  --resource-type "Microsoft.CognitiveServices/accounts" `
  --set properties.publicNetworkAccess="Enabled"  
#>

# Get the Azure Resource ID of your Custom OpenAI account.
$resourceId = az cognitiveservices account show `
  --name $serviceName `
  --resource-group $resourceGroup `
  --query id `
  -o tsv

# Add the custom subdomain (required for private endpoint)
az resource update `
  --ids $resourceId `
  --set properties.customSubDomainName=$customSubdomain

# Create Private Endpoint in $vnet (vnet-openai01)
az network private-endpoint create `
  --name "pe-$serviceName" `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --subnet $backendSubnet `
  --private-connection-resource-id $resourceId `
  --group-id "account" `
  --connection-name "conn-$serviceName"

# Get the DNS Zone Resource ID
$dnsZoneId = az network private-dns zone show `
  --name privatelink.openai.azure.com `
  --resource-group $hubResourceGroup `
  --query id `
  -o tsv

# Binds the Private Endpoint to the DNS zone
# So, that Azure injects the correct private IP mapping for your OpenAI endpoint.
# Without this, DNS resolution will fail or return public IP (which is disabled)
# The --zone-name parameter is required and should match the DNS zone name (e.g., privatelink.openai.azure.com).
# It must match the name of the Private DNS Zone you're associating — not just any label.
az network private-endpoint dns-zone-group create `
  --resource-group $resourceGroup `
  --endpoint-name "pe-$serviceName" `
  --name "dnszonegroup-$serviceName" `
  --private-dns-zone $dnsZoneId `
  --zone-name privatelink.openai.azure.com
```

### Step 9: Deploy a model inside the Azure OpenAI resource

```powershell
# Look at the available models in the region where $serviceName deployed
$models = az cognitiveservices account list-models `
  --name $serviceName `
  --resource-group $resourceGroup `
  | ConvertFrom-Json

$models |
    Where-Object { $_.lifecycleStatus -eq "GenerallyAvailable" -and $_.name -match "gpt-4" } |
    Select-Object name, version, format, lifecycleStatus |
    Format-Table -AutoSize
```

```powershell
# Deploy a model such as GPT-4o.
az cognitiveservices account deployment create `
  --name $serviceName `
  --resource-group $resourceGroup `
  --deployment-name "gpt4o-deployment" `
  --model-name "gpt-4o" `
  --model-version "2024-11-20" `
  --model-format OpenAI `
  --sku-capacity 1 `
  --sku-name Standard

<# Delete the model if needed as delete option is not visible in the Azure Portal

az cognitiveservices account deployment delete `
  --name $serviceName `
  --resource-group $resourceGroup `
  --deployment-name "gpt4o-deployment"
#>
```

```powershell
# Deploy a model such as gpt-4.1.
az cognitiveservices account deployment create `
  --name $serviceName `
  --resource-group $resourceGroup `
  --deployment-name "GPT-4.1" `
  --model-name "gpt-4.1" `
  --model-version "2025-04-14" `
  --model-format OpenAI `
  --sku-capacity 1 `
  --sku-name Standard

<# Delete the model if needed as delete option is not visible in the Azure Portal

az cognitiveservices account deployment delete `
  --name $serviceName `
  --resource-group $resourceGroup `
  --deployment-name "gpt41-deployment"
#>
```

```powershell
# verify the deployment
az cognitiveservices account deployment list `
  --name $serviceName `
  --resource-group $resourceGroup `
  -o table
```

### Step 10: Review the Keys and Endpoint

- Go to Resource Management -> Keys and Endpoint
- Note the followings to be used in Oracle to PostgreSQL migration
  - KEY 1
  - Endpoint 

### Step 10: Confirm the private endpoint status

```powershell
# Run a check to make sure the private endpoint is in "Approved" or "Succeeded" state (not "Pending")
az network private-endpoint show `
  --name "pe-$serviceName" `
  --resource-group $resourceGroup `
  --query "provisioningState" -o tsv
```

### Step 11: Update Token per minute (TPM)

- Migration requires TPM to be greated than 500K. You can set TPM in Powershell code. 
- Go to https://ai.azure.com/
- Search for Azure OpenAI service and open it
- Go to Deployments and select GPT-4.1
- Edit the configuration
- Update TPM to 500K
- Submit Changes

💡 **Note:**
Review for more details. [Deployment types for Azure AI Foundry Models](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-models/concepts/deployment-types)

### Step 11: Test from JumpboxVM01

```powershell
# Verify DNS resolution from the JumpboxVM01
# In this example, it will be "Resolve-DnsName AzOpenAIModern01.openai.azure.com"
Resolve-DnsName "$serviceName.openai.azure.com"

# It fails: DNS name does not exist

# Test connectivity from JumpboxVM01
# In this example, it will be "Test-NetConnection AzOpenAIModern01.openai.azure.com -Port 443"
Test-NetConnection "$serviceName.openai.azure.com" -Port 443

# It fails: WARNING: Name resolution of AzOpenAIModern01.openai.azure.com failed

<#
Follow the guidance here:

1. You create your Azure OpenAI resource with custom subdomain: AzOpenAIModern01CSD

2. Azure assigns your endpoint: https://azopenaimodern01csd.openai.azure.com/

This is the public-facing FQDN — used in API calls and SDKs.

3. You disable public access

    az resource update --set properties.publicNetworkAccess="Disabled"

Azure blocks public resolution and routing to azopenaimodern01csd.openai.azure.com. It can no longer be resolved via public DNS.

4. You then create a Private Endpoint and bind it to the Private DNS Zone

    privatelink.openai.azure.com

5. Azure injects a DNS record:

    azopenaimodern01csd.privatelink.openai.azure.com → 10.x.x.x (private IP)

This record is only visible inside your VNet (or peered VNets) that are linked to the Private DNS Zone.
#>

```

💡 Note:
You now have a fully configured Custom Open AI.
This setup can be used for migration testing, training, or integration validation with Azure PostgreSQL.