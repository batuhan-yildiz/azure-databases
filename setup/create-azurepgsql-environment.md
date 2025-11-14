# Create Azure Database for PostgreSQL Flexible Server Environment

This environment provisions an Azure Database for PostgreSQL Flexible Server instance within a workload VNet (spoke), configured for private access and integrated into a hub-and-spoke topology. You can use it  as the target system for schema and data migration from Oracle.

In this repository, the PostgreSQL Flexible Server is primarily used to:

- Receive migrated schemas and data from the Oracle source system
- Validate connectivity and DNS resolution from a jumpbox VM using private DNS zones
- Support testing and verification of converted objects and application compatibility
- Enable secure, scalable modernization workflows using Azure-native tooling


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
$resourceGroup="rg-pgsql03"
$vnet="vnet-pgsql03"
$backendSubnet="subnet-vnet-pgsql03-backend"
$backendSubnetNSG="nsg-subnet-vnet-pgsql03-backend"
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$jumpboxResourceGroup="rg-jumpbox"
$jumpboxVNet="vnet-jumpbox"
$serverName="azpgsqlprod01"
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for the the Azure PGSQL

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

### Step 4: Create the Azure PGSQL virtual network and subnet

```powershell
# Create a virtual network with a subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.6.0.0/16 `
  --subnet-name $backendSubnet `
  --subnet-prefix 10.6.0.0/24 `
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
# Peer pgsql03 to hub
az network vnet peering create `
  --name pgsql03-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

# Peer hub to pgsql03
az network vnet peering create `
  --name hub-to-pgsql03 `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  

# Peer pgsql03 to jumpbox
az network vnet peering create `
  --name pgsql03-to-jumpbox `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$jumpboxResourceGroup/providers/Microsoft.Network/virtualNetworks/$jumpboxVNet" `
  --allow-vnet-access

# Peer jumpbox to pgsql03
az network vnet peering create `
  --name jumpbox-to-pgsql03 `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  
```

### Step 7: Link the shared Private DNS Zone

Before creating the PostgreSQL server, link the shared private DNS zone (privatelink.postgres.database.azure.com) hosted in rg-hub to the VNet where your PostgreSQL server will reside.

> 💡 Why?
- Private endpoints rely on Azure Private DNS Zones to resolve the *.postgres.database.azure.com names to private IPs.
- This ensures your PostgreSQL server is reachable only within your VNets, not over the public internet.

```powershell
az network private-dns link vnet create `
  --resource-group $hubResourceGroup `
  --zone-name privatelink.postgres.database.azure.com `
  --name link-to-vnet-pgsql03-postgresql `
  --virtual-network "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --registration-enabled false
```

> ⚠️ Note:
The link must be created in the same resource group as the DNS zone (rg-hub), even if your PostgreSQL server is deployed in another group (rg-pgsql03).

### Step 8: Create the Azure Database for PostgreSQL Flexible Server

This command provisions a PostgreSQL 15 Flexible Server with private network connectivity, moderate compute size, and storage suitable for testing Oracle migrations.

⚠️ **Note:** Replace placeholder values like `<your-admin-username>` and `<your-admin-password>` with your own before running the commands.

```powershell
# Create Azure PGSQL
az postgres flexible-server create `
   --name $serverName `
   --resource-group $resourceGroup `
   --location $location `
   --vnet $vnet `
   --subnet $backendSubnet `
   --private-dns-zone "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/privateDnsZones/privatelink.postgres.database.azure.com" `
   --sku-name Standard_B2ms `
   --tier Burstable `
   --storage-size 64 `
   --storage-auto-grow Enabled `
   --version 15 `
   --admin-user <your-admin-username> `
   --admin-password '<your-admin-password>'
```
💡 Tip:
- Use Standard_B2ms for cost efficiency during testing.
- Storage auto-grow helps prevent issues during migration or data import.
- You can later scale compute or storage easily with Azure CLI or Portal.

### Step 9: Find the Private IP of Your PostgreSQL Server

Test network reachability from your Jumpbox or client VM within the same VNet or peered network.

```powershell
Test-NetConnection <postgresql_endpoint_from_azure_portal> -Port <postgresql_port>
```

```text
ComputerName     : <postgresql_endpoint_from_azure_portal>
RemoteAddress    : <postgresql_private_ip>
RemotePort       : <postgresql_port>
InterfaceAlias   : Ethernet
SourceAddress    : <jumpboxvm01_private_ip>
TcpTestSucceeded : True

✅ The TcpTestSucceeded: True result confirms successful connectivity over port <postgresql_port>.
In this example, the PostgreSQL server's private IP is <jumpboxvm01_private_ip>.
```

### Step 10: Create a database on Azure Database for PostreSQL for migration

- Go to Azure Portal
- Open Azure Database for PostgreSQL Felexible Server which you recently provisioned
- Expand Settings
- Click on Databases
- Add a new database
  - Name: customer_orders

💡 Note:
You now have a fully provisioned Azure Database for PostgreSQL Flexible Server with private network access.
This environment is ready for Oracle-to-PostgreSQL migration testing, data validation, and connectivity verification from your Jumpbox VM or other peered VNets.