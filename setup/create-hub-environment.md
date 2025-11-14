# Create Hub Environment (Shared Resources)

This document describes how to create the shared "hub" environment in Azure,
which includes the resource group, virtual network, and shared services used across multiple workloads (such as Jumpbox, Database Migration Service, PostgreSQL, etc.).

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
$location="westus2"
$resourceGroup="rg-hub"
$vnet="vnet-hub"
$backendSubnet="subnet-vnet-hub-backend"
$backendSubnetNSG="nsg-subnet-vnet-hub-backend"
$privateLinkSubnet="subnet-vnet-hub-privatelink"
$privateLinkSubnetNSG="nsg-subnet-vnet-hub-privatelink"
$logAnalytics="log-analytics-hub01"
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for the hub

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

💡 Note:
A Resource Group (RG) acts as a logical container for your resources.
The rg-hub group will store all shared network components that multiple workloads will depend on.

### Step 4: Create hub virtual network and subnet

```powershell
# Create VNet with initial subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.0.0.0/16 `
  --subnet-name $backendSubnet `
  --subnet-prefix 10.0.0.0/24 `
  --location $location

# Add additional subnet for Private Link
az network vnet subnet create `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --name $privateLinkSubnet `
  --address-prefix 10.0.1.0/24
```

### Step 5: Create network security group (NSG) and attach it to the subnet

```powershell
# Create a network security group
az network nsg create `
  --resource-group $resourceGroup `
  --name $backendSubnetNSG `
  --location $location

az network nsg create `
  --resource-group $resourceGroup `
  --name $privateLinkSubnetNSG `
  --location $location

# Attach network security group to virtual network subnet
az network vnet subnet update `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --name $backendSubnet `
  --network-security-group $backendSubnetNSG

az network vnet subnet update `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --name $privateLinkSubnet `
  --network-security-group $privateLinkSubnetNSG
```

### Step 6: Create log analytics

```powershell
# Create a Log Analytics Workspace
az monitor log-analytics workspace create `
  --resource-group $resourceGroup `
  --workspace-name $logAnalytics `
  --location $location
```

💡 Note:
The Log Analytics workspace allows you to collect telemetry from all workloads centrally.

### Step 7: Create private DNS zone

```powershell
# Create a shared Private DNS Zone for PostgreSQL servers
az network private-dns zone create `
  --resource-group $resourceGroup `
  --name privatelink.postgres.database.azure.com

# Create a shared Private DNS Zone for OpenAI servers
az network private-dns zone create `
  --resource-group $resourceGroup `
  --name privatelink.openai.azure.com
```

💡 Note:
- The Private DNS zone enables all services using Private Link like PostgreSQL and OpenAI to resolve domain names privately across VNets.  
- You do not need a separate Private DNS zone for each workload in most cases. One shared zone per service type is enough, and in many enterprise landing zones, you just create one shared DNS zone per service type in the hub. Let me break it down:

| Resource                                 | How many DNS Zones   | Why                                                                                                                      |
| ---------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Azure SQL Database / MI / Hyperscale** | 1 shared zone        | All private endpoints for SQL-based services can point to the same Private DNS zone: `privatelink.database.windows.net`. |
| **Azure PostgreSQL**                     | 1 shared zone        | Use `privatelink.postgres.database.azure.com` for all PostgreSQL servers.                                                |
| **Azure MySQL**                          | 1 shared zone        | Use `privatelink.mysql.database.azure.com` for all MySQL servers.                                                        |
| **Azure OpenAI**                         | 1 shared zone        | Use `privatelink.openai.azure.com` for all OpenAI services.                                                              |
| **Other services**                       | Only if required     | Only create a zone if you are using Private Endpoints and the service requires DNS resolution.                           |
