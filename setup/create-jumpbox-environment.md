# Create Jumpbox Environment

The Jumpbox is a virtual machine deployed in a workload VNet (spoke) that allows secure administrative access to resources in private networks.

In this repository, the Jumpbox will primarily be used to access:

- PostgreSQL databases deployed in private VNets (Oracle → PostgreSQL migrations)
- Any other private resources that do not have a public IP
- Azure CLI, PowerShell, and database management tools to manage workloads securely

> ⚠️ Note: The Jumpbox should not host production workloads. Its purpose is to provide controlled access to private network resources.


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
$resourceGroup="rg-jumpbox"
$vnet="vnet-jumpbox"
$backendSubnet="subnet-vnet-jumpbox-backend"
$backendSubnetNSG="nsg-subnet-vnet-jumpbox-backend"
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$publicIP="pip-JumpboxVM01"
$networkInterface="nic-JumpboxVM01"
$vmName="JumpboxVM01"
$osDisk="disk1-JumpboxVM01"
$rdpRuleName01 = "AllowRDP"
$sourceIP01 = "<your-home-or-office-public-ip>/32"   # Replace with your IP address
$priority01 = 100  # Lower number = higher priority
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for jumpbox

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

### Step 4: Create the Jumpbox virtual network and subnet

```powershell
# Create VNet with initial subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.1.0.0/16 `
  --subnet-name $backendSubnet `
  --subnet-prefix 10.1.0.0/24 `
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

### Step 6: Peer spoke vnets to hub

```powershell
# Peer jumpbox to hub
az network vnet peering create `
  --name jumpbox-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

# Peer hub to jumpbox
az network vnet peering create `
  --name hub-to-jumpbox `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  
```

### Step 7: Create network interface with private and public IP to be used by JumpboxVM01 Virtual Machine

```powershell
# Create public IP
az network public-ip create `
  --resource-group $resourceGroup `
  --name $publicIP `
  --sku Standard `
  --allocation-method Static `
  --location $location

# Create network interface with private and public IP
az network nic create `
  --resource-group $resourceGroup `
  --name $networkInterface `
  --vnet-name $vnet `
  --subnet $backendSubnet `
  --network-security-group $backendSubnetNSG `
  --public-ip-address $publicIP `
  --location $location
```

### Step 8: Link JumpboxVM01's VNet to the DNS zone

Linking JumpboxVM01 Virtual Machine's VNet to the private DNS zone is a critical step for enabling private name resolution of Azure PaaS services like your PostgreSQL Flexible Server. 

Private DNS Zone has veen created in hub resource group.

```powershell
# Link the VNet to the Private DNS Zone for PostgreSQL
az network private-dns link vnet create `
  --resource-group $hubResourceGroup `
  --zone-name privatelink.postgres.database.azure.com `
  --name link-to-vnet-jumpbox-postgresql `
  --virtual-network "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --registration-enabled false

# Link the VNet to the Private DNS Zone for OpenAI
az network private-dns link vnet create `
  --resource-group $hubResourceGroup `
  --zone-name privatelink.openai.azure.com `
  --name link-to-vnet-jumpbox-openai `
  --virtual-network "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --registration-enabled false

# Link the VNet to the Private DNS Zone for SQL Databases
az network private-dns link vnet create `
  --resource-group $hubResourceGroup `
  --zone-name privatelink.database.windows.net `
  --name link-to-vnet-jumpbox-azuresql `
  --virtual-network "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --registration-enabled false
```

### Step 9: Create a jumpbox vm

The Jumpbox VM is a small, secure virtual machine deployed in its own dedicated VNet, which is peered to hub VNet and and will be peered to other Vnets like the PostgreSQL VNet.

This design allows the Jumpbox to securely connect to private PostgreSQL servers and other workloads, while still leveraging the shared services (like DNS and monitoring) hosted in the hub.

It serves as a central administrative access point for performing management, migration, and troubleshooting tasks without exposing backend databases to the internet.

Typical use cases include:

- Installing and running VS Code or other migration tools (e.g., Oracle → PostgreSQL)
- Connecting securely to private PostgreSQL databases via internal IPs
- Using Azure CLI or PowerShell for workload provisioning and management
- Validating network connectivity across VNets
- Acting as a shared administrative VM for various Azure database workloads

💡 Note:

- The Jumpbox VNet should be peered to both the hub VNet (for shared services) and workload VNets (for direct connectivity).
- Use NSGs, Just-in-Time (JIT) VM access, or Azure Bastion to secure RDP access.
- The Jumpbox should remain lightweight and used only for administrative or testing tasks.

⚠️ **Note:** 
- Replace placeholder values like `<your-admin-username>` and `<your-admin-password>` with your own before running the commands.
- license-type Windows_Server: Save up to 49% with a license you already own using Azure Hybrid Benefit. You need to confirm you have an eligible Windows Server license with Software Assurance or Windows Server subscription to apply this Azure Hybrid Benefit.

```powershell
# Create the virtual machine
az vm create `
  --resource-group $resourceGroup `
  --name $vmName `
  --location $location `
  --nics $networkInterface `
  --image Win2022Datacenter `
  --size Standard_B2ms `
  --admin-username <your-admin-username> `
  --admin-password '<your-admin-password>' `
  --os-disk-name $osDisk `
  --license-type Windows_Server
```  

### Step 10: Allow RDP access to the Jumpbox

```powershell
# Create NSG rule to allow RDP
az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $backendSubnetNSG `
  --name $rdpRuleName01 `
  --priority $priority01 `
  --protocol Tcp `
  --source-address-prefixes $sourceIP01 `
  --destination-port-ranges 3389 `
  --description "Allow RDP access to Jumpbox from trusted IP"
```

💡 Note:
Capture the Public IP of the Jumbox to connect through RDP

### Step 11: Install VS Code in JumpboxVM01

  1. RDP to JumpboxVM01
  2. Go to the official download page from your jumpbox VM:
      https://code.visualstudio.com/download
  3. Download the "System Installer" for Windows:
      Look for: "Windows: System Installer and x64"
      User Installer: Installs VS Code only for the current user
      System Installer: Installs VS Code for all users on the system
  4. Run the installer as Administrator:
      This will install VS Code for all users on the VM
  5. Complete the installation:
      Accept defaults
      Enable 
        - Create a desktop icon
        - Add "Open with Code" action to Windows Explorer file context menu
        - Add "Open with Code" action to Windows Explorer directory context menu
        - Register Code as an editor for supported file types
        - Add to PATH

### Step 12: Install VS Code PostgreSQL extension in JumpboxVM01

- Open VS Code
- Go to Extensions View (Ctrl+Shift+X)
- Install PostgreSQL (Publisher Microsoft)
- Restart VS Code (recommended)
- Review the features
  - Connections
  - Query History
  - Migrations