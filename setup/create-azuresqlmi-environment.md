# Create Azure SQL Managed Instance Environment

This environment provisions an Azure SQL Managed Instance inside a workload VNet (spoke) with private access (private endpoint / private DNS), integrated into a hub‑and‑spoke topology. It’s designed to be the target platform for migrations from SQL Server anywhere (on‑prem, other clouds, or Arc‑enabled hosts), and to showcase SQL MI features, performance characteristics, and operational behaviors under realistic, network‑isolated conditions.

- MI Link – near real‑time, low‑downtime migration via Availability Groups
- Log Replay Service (LRS) – continuous log shipping for older versions
- Other methods – backup/restore or DMS for offline scenarios
- Showcase SQL MI features (compatibility, PaaS benefits, security)

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
$resourceGroup="rg-sqlmi01"
$vnet="vnet-sqlmi01"
$backendSubnet="subnet-vnet-sqlmi01-backend"
$backendSubnetNSG="nsg-subnet-vnet-sqlmi01-backend"
$routeTableName = "rt-$backendSubnet"
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$jumpboxResourceGroup="rg-jumpbox"
$jumpboxVNet="vnet-jumpbox"
$serverName="azsqlmimod01"
$adminUser = "azsqladmin"
$adminPassword = Read-Host -Prompt "Enter admin password:" -AsSecureString
$cred = New-Object System.Management.Automation.PSCredential($adminUser, $adminPassword)
$serverName="azsqlmidevtest01"
#$serverNameTemp="azsqlmimod01temp02"
$spName = "sqlmi-spm-azuresqlmidevtest01"

```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for the the Azure SQL MI

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

### Step 4: Create the Azure SQL MI virtual network and subnet

```powershell
# Create a virtual network with a subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.3.0.0/16 `
  --subnet-name $backendSubnet `
  --subnet-prefix 10.3.0.0/24 `
  --location $location

# Delegate the subnet before creating SQL MI
az network vnet subnet update `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --name $backendSubnet `
  --delegations Microsoft.Sql/managedInstances

# Create route table
az network route-table create `
  --resource-group $resourceGroup `
  --name $routeTableName `
  --location $location  

# Associate route table with subnet
az network vnet subnet update `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --name $backendSubnet `
  --route-table $routeTableName    
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
# Peer sqlmi01 to hub
az network vnet peering create `
  --name sqlmi01-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

# Peer hub to sqlmi01
az network vnet peering create `
  --name hub-to-sqlmi01 `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  

# Peer sqlmi01 to jumpbox
az network vnet peering create `
  --name sqlmi01-to-jumpbox `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$jumpboxResourceGroup/providers/Microsoft.Network/virtualNetworks/$jumpboxVNet" `
  --allow-vnet-access

# Peer jumpbox to sqlmi01
az network vnet peering create `
  --name jumpbox-to-sqlmi01 `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  
```

### Step 7: Link the shared Private DNS Zone

Before creating the Azure SQL MI server, link the shared private DNS zone (privatelink.database.windows.net) hosted in rg-hub to the VNet where your Azure SQL MI server will reside.

> 💡 Why?
- Private endpoints rely on Azure Private DNS Zones to resolve the *.database.windows.net names to private IPs.
- This ensures your Azure SQL MI server is reachable only within your VNets, not over the public internet.

```powershell
az network private-dns link vnet create `
  --resource-group $hubResourceGroup `
  --zone-name privatelink.database.windows.net `
  --name link-to-vnet-sqlmi01-azuresqlmi `
  --virtual-network "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --registration-enabled false
```

> ⚠️ Note:
The link must be created in the same resource group as the DNS zone (rg-hub), even if your Azure SQL MI server is deployed in another group (rg-sqlmi01).

### Step 8: Create the Azure SQL Managed Instance

This command provisions a Azure SQL MI with private network connectivity, moderate compute size, and storage.

```powershell
# If you use Entra authentication and not sure what to add to --external-admin-name and --external-admin-sid parameters, run the command below

$entraUser = az ad user list --output json | ConvertFrom-Json |
    ForEach-Object { $_ } |
    Where-Object { $_.displayName -like "<your_display_name>*" } |
    Select-Object displayName, userPrincipalName, id

# If you want to use SQL authentication, you need to set:
# --admin-user
# --admin-password
```

```powershell
# Create Azure SQL MI with Entra ID Authentication Only
az sql mi create `
   --name $serverName `
   --resource-group $resourceGroup `
   --location $location `
   --vnet-name $vnet `
   --subnet $backendSubnet `
   --license-type BasePrice `
   --capacity 4 `
   --storage 128 `
   --tier GeneralPurpose `
   --family Gen5 `
   --backup-storage-redundancy Zone `
   --collation SQL_Latin1_General_CP1_CI_AS `
   --timezone-id "Pacific Standard Time" `
   --zone-redundant false `
   --enable-ad-only-auth `
   --external-admin-principal-type "User" `
   --external-admin-name $entraUser.userPrincipalName `
   --external-admin-sid $entraUser.id   

   # --admin-user $adminUser `
   # --admin-password $adminPassword
```
💡 Tip:
- Use GP 4vCore.
- You can later scale compute or storage easily with Azure CLI or Portal.

### Step 9: Allows SQL traffic (1433) and Entra ID token flow (443)

```powershell
# NSG Rules on SQL MI Subnet
az network nsg rule list `
  --resource-group $resourceGroup `
  --nsg-name $backendSubnetNSG

# Allow SQL traffic from Jumpbox VNet
az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $backendSubnetNSG `
  --name Allow-SQL-Jumpbox `
  --priority 1000 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes 10.1.0.0/16 `
  --source-port-ranges '*' `
  --destination-port-ranges 1433 `
  --destination-address-prefixes 10.3.0.0/24

# Allow MI management traffic
az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $backendSubnetNSG `
  --name Allow-MI-Management `
  --priority 1100 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes AzureCloud `
  --destination-port-ranges 9000-9003

# Allow outbound HTTPS to Azure services
az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $backendSubnetNSG `
  --name Allow-Outbound-Azure `
  --priority 1200 `
  --direction Outbound `
  --access Allow `
  --protocol Tcp `
  --destination-address-prefixes AzureCloud `
  --destination-port-ranges 443

# Test the connection from JumpboxVM01
Test-NetConnection <host_from_azure_portal> -Port 1433
```

### Step 10: Create a service principal for SQL MI

```powershell
# Create SPN for Azure SQL MI
$spJson = az ad sp create-for-rbac `
    --name $spName `
    --skip-assignment `
    --output json

$sp = $spJson | ConvertFrom-Json

$appId  = $sp.appId
$secret = $sp.password
$tenant = $sp.tenant

Write-Host "SPN created:"
Write-Host "  App ID:      $appId"
Write-Host "  Secret:      $secret"
Write-Host "  Tenant ID:   $tenant"

# Assign Directory Reader role to the Service Principal

# Register the SPN with your SQL MI

```

