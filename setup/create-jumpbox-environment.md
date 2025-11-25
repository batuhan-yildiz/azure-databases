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

# Linux VM settings
$vmNameLinux="JumpboxVM02"
$networkInterfaceLinux="nic-JumpboxVM02"
$publicIPlinux="pip-JumpboxVM02"
$osDiskLinux="disk1-JumpboxVM02"
$adminUserLinux="azadmin"

$SSH_KEY_NAME="jumpboxvm02-key-ed"

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
# Create public IP (Windows)
az network public-ip create `
  --resource-group $resourceGroup `
  --name $publicIP `
  --sku Standard `
  --allocation-method Static `
  --location $location

# Create network interface with private and public IP (Windows VM)
az network nic create `
  --resource-group $resourceGroup `
  --name $networkInterface `
  --vnet-name $vnet `
  --subnet $backendSubnet `
  --network-security-group $backendSubnetNSG `
  --public-ip-address $publicIP `
  --location $location

# Create public IP (Linux)
az network public-ip create `
  --resource-group $resourceGroup `
  --name $publicIPLinux `
  --sku Standard `
  --allocation-method Static `
  --location $location

# Create network interface with private (Linux VM)
az network nic create `
  --resource-group $resourceGroup `
  --name $networkInterfaceLinux `
  --vnet-name $vnet `
  --subnet $backendSubnet `
  --public-ip-address $publicIPlinux `
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

### Step 9: Create a jumpbox and linux vm

#### Create a jumpbox vm

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
# Create the virtual machine (windows)
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

#### Generate an SSH key pair to securely connect to the Linux VM

💡 Note:
SSH (Secure Shell) is the most common method for securely accessing Linux VMs.
It encrypts your connection, ensuring that credentials and commands cannot be intercepted.
See [Microsoft Learn: Connect to a Linux VM](https://learn.microsoft.com/en-us/azure/virtual-machines/linux-vm-connect?tabs=Linux) for more details.

```powershell
# Create .ssh folder
New-Item -ItemType Directory -Path "$env:USERPROFILE\.ssh" -Force

# Generate SSH key pair
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\$SSH_KEY_NAME" -q -N '""'
```

| Parameter                 | Description                                                                                                                                                |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-t ed25519`              | Specifies the key type — **ED25519** is a modern, fast, and secure elliptic-curve algorithm.                                                               |
| `-f "$env:USERPROFILE\.ssh\$SSH_KEY_NAME"` | Defines the file path and name for your key pair.<br>📁 **Public key:** `~/.ssh/oraclevm01-key-ed.pub` <br>🔒 **Private key:** `~/.ssh/oraclevm01-key-ed` |
| `-q`                      | Runs quietly without printing progress messages.                                                                                                           |
| `-N '""'`                   | Creates a key with an **empty passphrase** (no password prompt when connecting).                                                                           |

> ⚠️ Security Note:
You can choose to set a passphrase (-N "your-passphrase") for extra protection if you’re storing your private key on a shared or less secure system.

#### Create a linux vm

```powershell
# Create the virtual machine (linux)
az vm create `
  --resource-group $resourceGroup `
  --name $vmNameLinux `
  --location $location `
  --nics $networkInterfaceLinux `
  --image Ubuntu2204 `
  --size Standard_B2ms `
  --admin-username $adminUserLinux `
  --ssh-key-values "$env:USERPROFILE\.ssh\$SSH_KEY_NAME.pub" `
  --authentication-type ssh `
  --security-type "TrustedLaunch" `
  --enable-agent true `
  --os-disk-name $osDiskLinux `
  --os-disk-size-gb 64
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

### Step 13: Connect to Linux VM

This guide explains how to connect to your Linux virtual machine from the JumpboxVM01 using SSH (Secure Shell).

The connection requires using the private key you generated earlier.

💡 Note:
SSH provides a secure, encrypted command-line connection to your VM.

#### Locate Your SSH Keys in Windows Powershell

When you generated the SSH key, two files were created.

```powershell
# List your SSH keys in Windows Powershell:
Get-ChildItem "$env:USERPROFILE\.ssh"
```

You should see both key files listed:

| File                    | Description                                       |
| ----------------------- | ------------------------------------------------- |
| `jumpboxvm02-key-ed`     | Private key - used to authenticate to your VM     |
| `jumpboxvm02-key-ed.pub` | Public key - uploaded to Azure during VM creation |

#### Copy the private key to Jumpbox01VM and secure the file

RDP to Jumpbox01VM

```powershell
# Create the .ssh folder if it doesn't exist
New-Item -ItemType Directory -Path "$env:USERPROFILE\.ssh" -Force
```

Copy the jumpboxvm02-key-ed file from your machine to Jumpbox01VM "$env:USERPROFILE\.ssh" folder.

Secure the Key File in Jumpbox01VM: You must set the correct permissions on the key to ensure SSH will accept it.

```powershell
# Run in PowerShell (Admin)
icacls "$env:USERPROFILE\.ssh\jumpboxvm02-key-ed" /inheritance:r /grant:r "${env:USERNAME}:R"
```

#### Connect to the Linux VM from Jumpbox01

Open Windows Powershell with Admin and run the following command

```powershell
#Use the following command to connect from Jumpbox:
ssh -i $env:USERPROFILE\.ssh\jumpboxvm02-key-ed azadmin@<linux-vm-private-ip>
# ssh -i $env:USERPROFILE\.ssh\jumpboxvm02-key-ed azadmin@10.1.0.5
```

💡 Note:
Replace `<oracle-vm-private-ip>` with your actual Oracle VM's private IP address

If you receive the following message, type **yes** then press enter. SSH will remember this host in your known_hosts file for future sessions.

```text
The authenticity of host '10.5.0.4 (10.5.0.4)' can't be established.
ED25519 key fingerprint is SHA256:GAqQ5DbLkkPox/5iNSwcG/vPCZxBcqgtAuOtfMEawvI.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```