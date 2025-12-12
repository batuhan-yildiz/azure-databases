# Create Azure SQL Database Environment

This environment provisions an Azure SQL Database inside a workload VNet (spoke) with private access, integrated into a hub‑and‑spoke topology. It’s designed to be the target platform for migrations from SQL Server anywhere (on‑prem, other clouds, or Arc‑enabled hosts), and to showcase SQL DB features, performance characteristics, and operational behaviors under realistic, network‑isolated conditions.

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
$resourceGroup="rg-sqldb01"
$vnet="vnet-sqldb01"
$backendSubnet="subnet-vnet-sqldb01-backend"
$backendSubnetNSG="nsg-subnet-vnet-sqldb01-backend"
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$jumpboxResourceGroup="rg-jumpbox"
$jumpboxVNet="vnet-jumpbox"
$adminUser = "azsqladmin"
$adminPassword = Read-Host -Prompt "Enter admin password:" -AsSecureString
$cred = New-Object System.Management.Automation.PSCredential($adminUser, $adminPassword)
$serverName="azsqldbdevtest01"
$databaseName="sqldb01"
$peName  = "pe-$serverName"
$nicName = "nic-pe-$serverName"
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for the the Azure SQL DB

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

### Step 4: Create the Azure SQL DB virtual network and subnet

```powershell
# Create a virtual network with a subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.8.0.0/16 `
  --subnet-name $backendSubnet `
  --subnet-prefix 10.8.0.0/24 `
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
# Peer sqlmi01 to hub
az network vnet peering create `
  --name sqldb01-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

# Peer hub to sqlmi01
az network vnet peering create `
  --name hub-to-sqldb01 `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  

# Peer sqlmi01 to jumpbox
az network vnet peering create `
  --name sqldb01-to-jumpbox `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$jumpboxResourceGroup/providers/Microsoft.Network/virtualNetworks/$jumpboxVNet" `
  --allow-vnet-access

# Peer jumpbox to sqlmi01
az network vnet peering create `
  --name jumpbox-to-sqldb01 `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access
```

### Step 7: Link the shared Private DNS Zone

Before creating the Azure SQL DB server, link the shared private DNS zone (privatelink.database.windows.net) hosted in rg-hub to the VNet where your Azure SQL DB will reside.

> 💡 Why?
- Private endpoints rely on Azure Private DNS Zones to resolve the *.database.windows.net names to private IPs.
- This ensures your Azure SQL DB is reachable only within your VNets, not over the public internet.

```powershell
az network private-dns link vnet create `
  --resource-group $hubResourceGroup `
  --zone-name privatelink.database.windows.net `
  --name link-to-vnet-sqldb01-azuresqldb `
  --virtual-network "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --registration-enabled false
```

> ⚠️ Note:
The link must be created in the same resource group as the DNS zone (rg-hub), even if your Azure SQL DB is deployed in another group (rg-sqldb01).

### Step 8: Create the Azure SQL Database

This command provisions a Azure SQL DB with private network connectivity, moderate compute size, and storage.

```powershell
# If you use Entra authentication and not sure what to add to --external-admin-name and --external-admin-sid parameters, run the command below

$entraUser = az ad user list --output json | ConvertFrom-Json |
    ForEach-Object { $_ } |
    Where-Object { $_.displayName -like "<your_display_name>*" } |
    Select-Object displayName, userPrincipalName, id

az sql server create `
  --name $serverName `
  --resource-group $resourceGroup `
  --location $location `
  --enable-ad-only-auth `
  --external-admin-principal-type "User" `
  --external-admin-name $entraUser.userPrincipalName `
  --external-admin-sid $entraUser.id

# If you want to use SQL authentication, you need to set:
# --admin-user
# --admin-password

# Disable public network access (private-only)
az sql server update `
   --name $serverName `
   --resource-group $resourceGroup `
   --set publicNetworkAccess=Disabled

az sql db create `
   --resource-group $resourceGroup `
   --server $serverName `
   --name $databaseName `
   --edition GeneralPurpose `
   --compute-model Serverless `
   --family Gen5 `
   --min-capacity 0.5 `
   --capacity 2 `
   --auto-pause-delay 60 `
   --zone-redundant false

az network private-endpoint create `
  --name $peName `
  --resource-group $resourceGroup `
  --location $location `
  --vnet-name $vnet `
  --subnet $backendSubnet `
  --nic-name $nicName `
  --private-connection-resource-id $(az sql server show -n $serverName -g $resourceGroup --query id -o tsv) `
  --group-id sqlServer `
  --connection-name "connection-$serverName"

# Attach DNS Zone Group to Your Private Endpoint
az network private-endpoint dns-zone-group create `
  --resource-group $resourceGroup `
  --endpoint-name $peName `
  --name "pdzg-$serverName" `
  --private-dns-zone "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/privateDnsZones/privatelink.database.windows.net" `
  --zone-name privatelink.database.windows.net
```

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
  --destination-address-prefixes 10.8.0.0/24

# Test the connection from JumpboxVM01
Test-NetConnection azsqldbdevtest01.privatelink.database.windows.net -Port 1433
```