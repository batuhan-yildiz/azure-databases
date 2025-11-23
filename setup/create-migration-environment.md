# Create Migration Environment

This environment provisions shared general migration resources used by all migration scenarios.

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
$resourceGroup="rg-migration"
$vnet="vnet-migration"
$backendSubnet="subnet-vnet-migration-backend"
$backendSubnetNSG="nsg-subnet-vnet-migration-backend"
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$jumpboxResourceGroup="rg-jumpbox"
$jumpboxVNet="vnet-jumpbox"
$storageAccount="samodernization01"
$adfName="adf-modernization01"
$kvName="kv-modernization01"
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
# Peer migration to hub
az network vnet peering create `
  --name migration-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

# Peer hub to migration
az network vnet peering create `
  --name hub-to-migration `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  

# Peer migration to jumpbox
az network vnet peering create `
  --name migration-to-jumpbox `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$jumpboxResourceGroup/providers/Microsoft.Network/virtualNetworks/$jumpboxVNet" `
  --allow-vnet-access

# Peer jumpbox to migration
az network vnet peering create `
  --name jumpbox-to-migration `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  
```

### Step 7: Register the reguired providers and add extension in your subscription 

```powershell
# Register the Provider for Storage
az provider register --namespace Microsoft.Storage

# Check registration status (It should return **Registered**)
az provider show --namespace Microsoft.Storage --query "registrationState"

# Install data factory extension
az extension add --name datafactory

# Register the Provider for DataFactory
az provider register --namespace Microsoft.DataFactory

# Check registration status (It should return **Registered**)
az provider show --namespace Microsoft.DataFactory --query "registrationState"

# Register the Provider for KeyVault
az provider register --namespace Microsoft.KeyVault

# Check registration status (It should return **Registered**)
az provider show --namespace Microsoft.KeyVault --query "registrationState"
```

### Step 8: Create storage account

Used for staging files, schema export outputs, large table CSV/Parquet staging, logs, and artifacts.

ADF pipelines often stage data temporarily (especially for large Oracle tables). Also used for logging & backup dumps.

```powershell
# Create storage account
az storage account create `
  --name $storageAccount `
  --resource-group $resourceGroup `
  --location $location `
  --sku Standard_LRS `
  --kind StorageV2 `
  --access-tier Hot `
  --enable-hierarchical-namespace true `
  --public-network-access Enabled  
```

### Step 9: Create Azure Data Factory (ADF) for migration

ADF orchestrates full load, incremental load, and cutover tasks for any source-to-target migration.

```powershell
# Create ADF
az datafactory create `
  --resource-group $resourceGroup `
  --factory-name $adfName `
  --location $location
```

### Step 10: Create keyvault

```powershell
# create key vault
az keyvault create `
  --name $kvName `
  --resource-group $resourceGroup `
  --location $location `
  --sku standard
```

### Step 11: ADF authentication against Key Vault

- Your current identity may NOT have permission to write secrets into the Key Vault.
- There might be an Azure RBAC / Key Vault policy issue.
- By default, ADF doesn’t have its own identity unless you explicitly assign one. Without an identity, it can’t authenticate against Key Vault. 
- Once enabled, ADF can use this identity to authenticate securely to other Azure services (like Key Vault) without needing credentials or secrets stored in code.

#### Enables a System-Assigned Managed Identity for your Data Factory.

- You cannot use az datafactory update with --set identity.type=SystemAssigned. 
- The CLI command does not support updating the identity.    

Use the following method to enable system-assigned identity
  - Go to your Data Factory resource (adf-modernization01).
  - In the left menu, select Identity → System assigned.
  - Turn it On, click Save.
  - Wait until the managed identity is created.

```powershell
# Verify systemassigned identity. Review type
az datafactory show `
    --resource-group $resourceGroup `
    --name $adfName `
    --query identity

# Get the ADF principal ID
$adfPrincipalId = az datafactory show `
--resource-group $resourceGroup `
--name $adfName `
--query "identity.principalId" -o tsv

# Check RBAC mode. If not enabled, give permission to ADF to read secrets in your Key Vault.
$rbacEnabled = az keyvault show `
    --name $kvName `
    --resource-group $resourceGroup `
    --query properties.enableRbacAuthorization `
    --output tsv

if ($rbacEnabled -eq "true") {
    Write-Host "RBAC authorization is enabled on Key Vault '$kvName'. Access policies cannot be set. Use RBAC role assignments instead."
} else {
    Write-Host "RBAC authorization is NOT enabled. Setting access policy..."
    az keyvault set-policy `
        --name $kvName `
        --object-id $adfPrincipalId `
        --secret-permissions get list
    Write-Host "Access policy set successfully for ADF principal."
}

# Get key vault resource ID
$kvResourceId = az keyvault show --name $kvName --query id -o tsv

# Assign Key Vault permissions
az role assignment create `
--assignee $adfPrincipalId `
--role "Key Vault Secrets User" `
--scope $kvResourceId
```


