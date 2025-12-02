# Create Windows and SQL Arc Environment

The HyperV server is a virtual machine deployed in a workload VNet (spoke) that allows secure administrative access to resources in private networks.

In this repository, this VM will primarily be used to access:

- Hyper-V and nested VMs
- Windows and SQL Arc

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
$resourceGroup="rg-onprem"
$vnet="vnet-onprem"
$backendSubnet="subnet-vnet-onprem-backend"
$backendSubnetNSG="nsg-subnet-vnet-onprem-backend"
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$jumpboxResourceGroup="rg-jumpbox"
$jumpboxVNet="vnet-jumpbox"
$publicIP="pip-HyperHostVM01"
$networkInterface="nic-HyperHostVM01"
$vmName="HyperHostVM01"
$osDisk="disk1-HyperHostVM01"
$dataDisk="disk2-HyperHostVM01"
$rdpRuleName01 = "AllowRDP"
$sourceIP01 = "<your-home-or-office-public-ip>/32"   # Replace with your IP address
$priority01 = 100  # Lower number = higher priority

# HyperV Configuration
$switchName = "ExternalSwitch"
$hostNic = "Ethernet"

$nestedVMs          = @(
    @{Name="DC01";   IP="10.2.0.50"}, 
    @{Name="Node01"; IP="10.2.0.51"}, 
    @{Name="Node02"; IP="10.2.0.52"}, 
    @{Name="Node03"; IP="10.2.0.53"}, 
    @{Name="Node04"; IP="10.2.0.54"}, 
    @{Name="Node05"; IP="10.2.0.55"} 
)
$isoPath = "C:\Products\WindowsServer2022.iso"
$vmStorePath = "E:\HyperV\Virtual Hard Disks"
$vmConfigStorePath = "E:\HyperV\Virtual Machines"

$gatewayNestedVMs = "10.2.0.1"
$prefixLen = 24

# Nested VMs General
$vmMemoryDC = 4GB
$vmMemoryNode = 6GB
$vmProcessorCount = 2
$vmVhdSizeGB = 80GB   # size for each nested VM VHDX
$localAdminUser = "Administrator"  # will be used when automating into guest
$plainPassword = Read-Host -Prompt "Enter nested VMs local Administrator password (will be used for PowerShell Direct)" -AsSecureString
$cred = New-Object System.Management.Automation.PSCredential($localAdminUser, $plainPassword)
$domainName = "contoso.local"

```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for HyperHostVM01

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

### Step 4: Create the HyperHostVM01 virtual network and subnet

```powershell
# Create VNet with initial subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.2.0.0/16 `
  --subnet-name $backendSubnet `
  --subnet-prefix 10.2.0.0/24 `
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
# Peer onprem to hub
az network vnet peering create `
  --name onprem-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

# Peer hub to onprem
az network vnet peering create `
  --name hub-to-onprem `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access

# Peer onprem to jumpbox
az network vnet peering create `
  --name onprem-to-jumpbox `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$jumpboxResourceGroup/providers/Microsoft.Network/virtualNetworks/$jumpboxVNet" `
  --allow-vnet-access `
  --allow-forwarded-traffic

# Peer jumpbox to onprem
az network vnet peering create `
  --name jumpbox-to-onprem `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access `
  --allow-forwarded-traffic
```

### Step 7: Create network interface with private and public IP to be used by HyperHostVM01 Virtual Machine

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

### Step 8: Create a HyperHostVM01 vm

⚠️ **Note:** 
- Replace placeholder values like `<your-admin-username>` and `<your-admin-password>` with your own before running the commands.
- license-type Windows_Server: Save up to 49% with a license you already own using Azure Hybrid Benefit. You need to confirm you have an eligible Windows Server license with Software Assurance or Windows Server subscription to apply this Azure Hybrid Benefit.
- Standard_D16s_v4: 16 vCPUs, 64GB RAM

```powershell
# Create the virtual machine
az vm create `
  --resource-group $resourceGroup `
  --name $vmName `
  --location $location `
  --nics $networkInterface `
  --image Win2022Datacenter `
  --size Standard_D16s_v4 `
  --admin-username <your-admin-username> `
  --admin-password '<your-admin-password>' `
  --os-disk-name $osDisk `
  --license-type Windows_Server

# Create secondary disk
az disk create `
  --resource-group $resourceGroup `
  --name $dataDisk `
  --size-gb 512 `
  --sku Premium_LRS

# Attach the secondary disk
az vm disk attach `
  --resource-group $resourceGroup `
  --vm-name $vmName `
  --name $dataDisk
```  

### Step 9: Allow RDP access to the HyperHostVM01

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

### Step 10: Configure HyperHostVM01

RDP to HyperHostVM01 VM

#### Initialize and Format the Disk Inside the Windows VM

```powershell
# List disks
Get-Disk

# Initialize the new disk (replace <disknumber> with the number you saw in Get-Disk - RAW partition style)
Initialize-Disk -Number <disknumber> -PartitionStyle GPT

# Create partition
New-Partition -DiskNumber <disknumber> -UseMaximumSize -DriveLetter E

# Format as NTFS
Format-Volume -DriveLetter E -FileSystem NTFS -NewFileSystemLabel "HyperVData" -Confirm:$false
```

#### Configure Hyper-V to Use E:\ for VHDX Storage

```powershell
# Configure Hyper-V to Use E:\ for VHDX Storage
Set-VMHost -VirtualHardDiskPath $vmStorePath `
           -VirtualMachinePath $vmConfigStorePath

# Create folders
if (-not (Test-Path $vmStorePath)) 
  { New-Item -ItemType Directory -Path $vmStorePath | Out-Null }
if (-not (Test-Path $vmConfigStorePath)) 
  { New-Item -ItemType Directory -Path $vmConfigStorePath | Out-Null }
```

#### Install Hyper-V and configure

```powershell
# Install Hyper-V role and restart the VM
Install-WindowsFeature -Name Hyper-V -IncludeAllSubFeature -IncludeManagementTools -Restart:$false

# Enable IP routing in Windows so host can forward traffic
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "IPEnableRouter" -Value 1

# Create External vSwitch
if (-not (Get-VMSwitch -Name $switchName -ErrorAction SilentlyContinue)) {
    New-VMSwitch -Name $switchName `
                 -NetAdapterName $hostNIC `
                 -SwitchType External `                 
                 -AllowManagementOS $true
    Write-Host "Created external switch: $switchName"
} `
else { Write-Host "Switch $switchName already exists" }

# Confirm the switch
Get-VMSwitch | Format-Table Name, SwitchType, NetAdapterInterfaceDescription
```

#### Create the VMs

```powershell
foreach ($vm in $nestedVMs) {

    $nestedVMName   = $vm.Name
    $vhdPath = "$vmStorePath\$nestedVMName  .vhdx"
    $MemoryStartupBytes = $vmMemoryDC

    if ($nestedVMName   -ne "DC01") { $MemoryStartupBytes = $vmMemoryNode }


    # Create VM
    New-VM -Name $nestedVMName   `
           -MemoryStartupBytes $MemoryStartupBytes `
           -BootDevice VHD `
           -SwitchName $switchName `
           -Generation 2 `
           -NewVHDPath $vhdPath `
           -NewVHDSizeBytes $vmVhdSizeGB

    # CPU
    Set-VMProcessor -VMName $nestedVMName   -Count $vmProcessorCount

    # Attach ISO for installation
    Add-VMDvdDrive -VMName $nestedVMName   -Path $isoPath

    Write-Host "VM $nestedVMName   created."
}
```

#### Complete OS installation in each VM console

```powershell
# Start each VM to install the OS

# Start the VM and Complete OS installation one by one
Start-VM -Name $nestedVMs[0].Name
Start-VM -Name $nestedVMs[1].Name
Start-VM -Name $nestedVMs[2].Name
Start-VM -Name $nestedVMs[3].Name
Start-VM -Name $nestedVMs[4].Name
Start-VM -Name $nestedVMs[5].Name
```
#### Change computer name

```powershell
# Change the computer name
foreach ($vm in $nestedVMs) {

    $nestedVMName   = $vm.Name

    Invoke-Command -VMName $nestedVMName   -Credential $cred -ScriptBlock {
        param($newName)

        Rename-Computer -NewName $newName -Force -Restart
    } -ArgumentList $nestedVMName  

    Write-Host "VM $nestedVMName   computer name updated."
}  
```

#### Configure network inside DC01

```powershell
Invoke-Command -VMName $nestedVMs[0].Name -Credential $cred -ScriptBlock {
    param($ip, $prefix, $gateway)
    
    $if = Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.Name -like "*Ethernet*" } | Select-Object -First 1
    if ($null -eq $if) { throw "Cannot find network adapter inside guest." }
    $ifAlias = $if.Name

    # Remove DHCP IPv4
    Get-NetIPAddress -InterfaceAlias $ifAlias -AddressFamily IPv4 | Remove-NetIPAddress -Confirm:$false -ErrorAction SilentlyContinue

    # Set static IP
    New-NetIPAddress -InterfaceAlias $ifAlias -IPAddress $ip -PrefixLength $prefix -DefaultGateway $gateway

    # Set DNS to self
    Set-DnsClientServerAddress -InterfaceAlias $ifAlias -ServerAddresses $ip
} -ArgumentList $nestedVMs[0].IP, $prefixLen, $gatewayNestedVMs
```

#### Promote DC01 to Domain Controller

```powershell
$domainNetbios = ($domainName.Split('.')[0])

Invoke-Command -VMName $nestedVMs[0].Name -Credential $cred -ScriptBlock {
    param($domainName, $domainNetbios, $plainPassword)

    Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

    # convert password string to secure string
    $securePwd = ConvertTo-SecureString $plainPassword -AsPlainText -Force

    # Promote to DC (creates forest)
    Install-ADDSForest -DomainName $domainName `
                       -DomainNetbiosName $domainNetbios `
                       -SafeModeAdministratorPassword $securePwd `
                       -InstallDns `
                       -Force
} -ArgumentList $domainName, $domainNetbios, $plainPassword
```

#### Connect to DC01 Nested VM and create AD users and groups

```powershell
# Group and users to create
# Run this script **inside DC01** directly.
Import-Module ActiveDirectory

$domainName = "contoso.local"
$plainPassword = Read-Host -Prompt "Enter domain user password" -AsSecureString

$groupName = "SQL Admins"
$users = @(
    @{Name="sqluser"; Password=$plainPassword},
    @{Name="sqlservice"; Password=$plainPassword},
    @{Name="sqladmin"; Password=$plainPassword}
)

# Create group
if (-not (Get-ADGroup -Filter {Name -eq $groupName} -ErrorAction SilentlyContinue)) {
    New-ADGroup -Name $groupName -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=contoso,DC=local"
    Write-Host "Group Name '$groupName' created."
} else {
        Write-Host "Group Name '$groupName' already exists."
}

# Create users
foreach ($user in $users) {
    $samName = $user.Name
    $securePwd = $user.Password

    if (-not (Get-ADUser -Filter "SamAccountName -eq '$samName'" -ErrorAction SilentlyContinue)) {

        New-ADUser -Name $samName `
                   -SamAccountName $samName `
                   -AccountPassword $securePwd `
                   -Enabled $true `
                   -Path "CN=Users,DC=contoso,DC=local"
        Write-Host "User '$samName' created."
    } else {
        Write-Host "User '$samName' already exists."
    }
}

# Add sqladmin to group
Add-ADGroupMember -Identity $groupName -Members "sqladmin"
```

#### Configure network for Nodes (Node01 → Node05)

```powershell
foreach ($vm in $nestedVMs) {

    $nestedVMName   = $vm.Name
    $nestedVMIP   = $vm.IP

    if ($nestedVMName -ne "DC01") { 

        Invoke-Command -VMName $nestedVMName -Credential $cred -ScriptBlock {
            param($ip, $prefix, $gateway, $dns)
            $if = Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.Name -like "*Ethernet*" } | Select-Object -First 1
            if ($null -eq $if) { throw "Cannot find network adapter inside guest." }
            $ifAlias = $if.Name

            # Remove DHCP IP
            Get-NetIPAddress -InterfaceAlias $ifAlias -AddressFamily IPv4 | Remove-NetIPAddress -Confirm:$false -ErrorAction SilentlyContinue

            # Set static IP
            New-NetIPAddress -InterfaceAlias $ifAlias -IPAddress $ip -PrefixLength $prefix -DefaultGateway $gateway

            # Set DNS to DC01
            Set-DnsClientServerAddress -InterfaceAlias $ifAlias -ServerAddresses $dns
        } -ArgumentList $nestedVMIP, $prefixLen, $gatewayNestedVMs, $nestedVMs[0].IP
    }
}
```

#### Join Nodes to Domain

```powershell
$domainAdminUPN= "Administrator@$domainName"

foreach ($vm in $nestedVMs) {

    $nestedVMName   = $vm.Name
    $nestedVMIP   = $vm.IP

    if ($nestedVMName -ne "DC01") { 

        Invoke-Command -VMName $nestedVMName -Credential $cred -ScriptBlock {
            param($domainName, $domainAdminUPN, $plainPassword)
            $domainCred = New-Object System.Management.Automation.PSCredential($domainAdminUPN, $plainPassword)

            Add-Computer -DomainName $domainName -Credential $domainCred -Restart
        } -ArgumentList $domainName, $domainAdminUPN, $plainPassword

    }
}
```

#### Add Domain Group to Local Administrators

- Connect to nested VM with local administrator account .\Administrator
- Run the following configurations inside the nested VM

```powershell
# Add domain group to local Administrator group
# Run this script **inside Node** directly through Powershell.
$domainGroup = "SQL Admins"
$localGroup = [ADSI]"WinNT://./Administrators,group"
$localGroup.Add("WinNT://contoso/$domainGroup,group")

# Enable inbound ICMP (ping) on Node
# Enable File and Printer Sharing (Echo Request - ICMPv4-In)
Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"
# Optional: check rule
Get-NetFirewallRule -DisplayGroup "File and Printer Sharing" | ft DisplayName,Enabled

# Enable SQL Port
New-NetFirewallRule -DisplayName "SQL Server 1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
```

- logout and login to Node as domain user sqladmin which is local admin on Node










#### Install Windows Cluster

Delegate permissions on the Computers container in Active Directory.
- contoso\sqladmin → used to create the Windows Failover Cluster
- contoso\sqlservice → used by SQL service

Connect to DC01 and run the code inside DC01 to delegate the permission through Powershell

```powershell
<#
.SYNOPSIS
    Delegate Active Directory permissions for SQL Server FCI setup.
.DESCRIPTION
    This script grants necessary AD permissions to the domain accounts:
    - contoso\sqladmin → used to create the Windows Failover Cluster
    - contoso\sqlservice → used for SQL Server FCI service
    Both accounts need Create Computer Objects and Read all properties
    on the Computers container.
    Run this script directly on DC01.
#>

Import-Module ActiveDirectory

# --- Accounts that need delegation ---
$accounts = @(
    "contoso\sqladmin",
    "contoso\sqlservice"
)

# --- Container where cluster and SQL objects will be created ---
$containerDN = "CN=Computers,DC=contoso,DC=local"

# Get the schema object for computer class
$computerSchema = Get-ADObject -LDAPFilter "(lDAPDisplayName=computer)" -SearchBase "CN=Schema,CN=Configuration,DC=contoso,DC=local" -Properties schemaIDGUID

# Convert byte array to GUID
$guidBytes = $computerSchema.schemaIDGUID
[guid]::New($guidBytes)
# bf967a86-0de6-11d0-a285-00aa003049e2

# --- GUID for Computer object class in AD schema ---
$computerClassGuid = [GUID]"bf967a86-0de6-11d0-a285-00aa003049e2"

Write-Host "Delegating permissions on $containerDN ..." -ForegroundColor Yellow

# Get the container
$obj = Get-ADObject -Identity $containerDN -Properties nTSecurityDescriptor

# Loop through accounts
foreach ($acct in $accounts) {
    Write-Host "`nProcessing account: $acct" -ForegroundColor Cyan

    # Convert to NTAccount
    $idRef = New-Object System.Security.Principal.NTAccount($acct)

    # Build the rights properly
    $rights = [System.DirectoryServices.ActiveDirectoryRights]::CreateChild
    $rights = $rights -bor [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty

    # Create the ACE
    $ace = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
        $idRef,
        $rights,
        [System.Security.AccessControl.AccessControlType]::Allow,
        $computerClassGuid,
        "All"
    )

    # Add ACE to security descriptor
    $sd = $obj.nTSecurityDescriptor
    $sd.AddAccessRule($ace)

    # Commit changes
    Set-ADObject -Identity $containerDN -Replace @{nTSecurityDescriptor = $sd}

    Write-Host "Permissions applied to $acct" -ForegroundColor Green
}
```

Connect to Node01, Node02 and Node03 and run the code inside the nested VMs to Install Failover Clustering feature through Powershell

```powershell
# Run inside each node
Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools -Verbose
```

Connect to Node01 and run the code inside the nested VM to setup the cluster including Node01, Node02 nodes. Once the setup is complete, open Failover Cluster Manager in Node01 and review it.

```powershell
# Variables
$ClusterName = "Cluster01"
$ClusterIP   = "10.2.100.51"

# Runs Microsoft’s required cluster validation tests: 
# Network tests, Storage tests, System configuration, Domain checks, Node heartbeat checks
Test-Cluster -Node Node01, Node02, Node03

# Create failover cluster
New-Cluster `
    -Name $ClusterName `
    -Node Node01, Node02, Node03 `
    -StaticAddress $ClusterIP `
    -NoStorage

# Check the setup state. If it says Available, then it is not installed.
Get-WindowsFeature Failover-Clustering

# Enable firewall rules (inside all nodes):
Get-NetFirewallRule | Select-Object DisplayGroup -Unique | Sort-Object DisplayGroup
Enable-NetFirewallRule -DisplayGroup "Failover Clusters"
```

#### Create shared disks for cluster

Create a new disk and attach it to DC01 (Run on HostVM Powershell)

```powershell
$dcVhdShared = Join-Path $vmStorePath $dcVmName"Shared.vhdx"
$dcVhdSharedSizeGB = 256

# Create VHDX if it does not exist
if (-not (Test-Path $dcVhdShared)) {
    New-VHD -Path $dcVhdShared -SizeBytes ($dcVhdSharedSizeGB * 1GB) -Dynamic
    Write-Host "Created VHDX: $dcVhdShared"
} else {
    Write-Host "VHDX already exists: $dcVhdShared"
}

# Attach to DC01 on SCSI controller (safe when VM is running)
Add-VMHardDiskDrive -VMName $dcVmName -ControllerType SCSI -Path $dcVhdShared
Write-Host "Attached $dcVhdShared to VM $dcVmName"
```

Initialize the disk in DC01 (Run inside DC01 as Administrator)

```powershell
# Find uninitialized disks (new VHDX)
$rawDisks = Get-Disk | Where-Object PartitionStyle -Eq 'RAW'
if ($rawDisks.Count -eq 0) {
    Write-Host "No RAW disks found. If VHDX was attached, ensure disk shows up in Disk Management."
} else {
    foreach ($d in $rawDisks) {
        Write-Host "Initializing disk number $($d.Number) size $([math]::Round($d.Size/1GB)) GB"
        Initialize-Disk -Number $d.Number -PartitionStyle GPT -PassThru |
            New-Partition -UseMaximumSize -AssignDriveLetter |
            Format-Volume -FileSystem NTFS -NewFileSystemLabel "ClusterShared" -Confirm:$false
    }
}

# Create folder for iSCSI virtual disk files: E:\Shared Disks (if you store virtual disk files on E:)
$sharedFolder = "E:\Shared Disks"
if (-not (Test-Path $sharedFolder)) { New-Item -Path $sharedFolder -ItemType Directory -Force }
Write-Host "Shared folder: $sharedFolder"

# Install iSCSI Target Server role
Install-WindowsFeature -Name FS-iSCSITarget-Server -IncludeManagementTools -Verbose

# Define paths for virtual disks and sizes
$vdQuorum = Join-Path $sharedFolder "Cluster01Quorum.vhdx"
$vdData  = Join-Path $sharedFolder "Cluster01Shared01.vhdx"

# Create iSCSI virtual disks (file-backed)
New-IscsiVirtualDisk -Path $vdQuorum -Size 4GB -Description "Cluster01 Quorum Disk"
New-IscsiVirtualDisk -Path $vdData -Size 32GB -Description "Cluster01 Shared Disk 01"

# Create an iSCSI target
$targetName = "Target-Cluster01"
New-IscsiServerTarget -TargetName $targetName -InitiatorId @("IPAddress:10.2.100.12","IPAddress:10.2.100.13","IPAddress:10.2.100.14")
Get-IscsiServerTarget -TargetName $targetName
<# 
- If you want to add another IP, you need to use Set-IscsiServerTarget to update the InitiatorId list.
- However, Set-IscsiServerTarget replaces the entire list, so you need to include all previous IPs along with the new one.

Set-IscsiServerTarget -TargetName "Target-Cluster01" -InitiatorId @(
    "IPAddress:10.2.100.12",
    "IPAddress:10.2.100.13",
    "IPAddress:10.2.100.14")
#>

# Map virtual disks to target
Add-IscsiVirtualDiskTargetMapping -TargetName $targetName -Path $vdQuorum
Add-IscsiVirtualDiskTargetMapping -TargetName $targetName -Path $vdData

<# 
Optionally restrict initiators (recommended) — get IQN from Node01/Node02 (see next phase)
- Example placeholder (replace with real IQNs you will gather from Node01/Node02):
- $initiators = @("iqn.1991-05.com.microsoft:node01", "iqn.1991-05.com.microsoft:node02")
- foreach ($id in $initiators) { Grant-IscsiServerTarget -TargetName $targetName -InitiatorId $id }
#>
```

#### Configure initiators on Node01, Node02 and Node03 (run on each node)

```powershell
# Ensure Microsoft iSCSI Initiator service runs
Set-Service -Name MSiSCSI -StartupType Automatic
Start-Service -Name MSiSCSI

# Add the target portal (DC01 ip is 10.2.100.11 — adjust if different):
$targetPortalIP = "10.2.100.11"
New-IscsiTargetPortal -TargetPortalAddress $targetPortalIP -ErrorAction SilentlyContinue

# Discover targets
Get-IscsiTarget

# Connect to all discovered targets (persist across reboots)
Get-IscsiTarget | Where-Object { $_.NodeAddress -ne $null } | ForEach-Object {
    $node = $_
    Connect-IscsiTarget -NodeAddress $node.NodeAddress -IsPersistent $true -ErrorAction Stop
    Write-Host "Connected to iSCSI target: $($node.NodeAddress)"
}

# List sessions
Get-IscsiSession

# List new disks (usually appear as RAW)
Get-Disk | Where-Object PartitionStyle -eq 'RAW' | Format-Table Number, FriendlyName, Size, PartitionStyle
```

#### Add the quorum disk (run on Node01)

- Initialize (format) the disk on Node01, add to cluster
- Important cluster rule: For cluster disks, pick one node to initialize and format the disk (Node01). 
- After formatting, bring disk online and then add to the cluster. Cluster will manage ownership.

```powershell
# Show all disks and find the one that is the iSCSI LUN you want (e.g. 4GB quorum)
Get-Disk | Sort-Object Size | Format-Table Number, FriendlyName, Size, OperationalStatus, PartitionStyle

# Example: assume quorum disk is Disk 1 (replace DiskNumber ($quorumDiskNumber) accordingly)
$quorumDiskNumber = (Get-Disk | Where-Object { $_.Size -le (5 * 1GB) -and $_.PartitionStyle -eq 'RAW' } | Select-Object -First 1).Number
Write-Host "Quorum disk number is $quorumDiskNumber"

# Bring disk online (if offline), initialize, create partition, format (no drive letter)
# Check the disk state
Get-Disk -Number $quorumDiskNumber | Select Number, OperationalStatus, IsOffline, IsReadOnly, PartitionStyle

Set-Disk -Number $quorumDiskNumber -IsOffline $false
Set-Disk -Number $quorumDiskNumber -IsReadOnly $false

Initialize-Disk -Number $quorumDiskNumber -PartitionStyle GPT

$part = New-Partition -DiskNumber $quorumDiskNumber -UseMaximumSize -AssignDriveLetter
Format-Volume -Partition $part -FileSystem NTFS -NewFileSystemLabel "ClusterQuorum" -Confirm:$false

# Add the disk to the Windows Failover Cluster

# Note the disk number 
Get-ClusterAvailableDisk | Format-Table -AutoSize

# Add the disk to cluster
$diskToAdd = Get-ClusterAvailableDisk | Where-Object { ($_.Number -eq 1) }

Add-ClusterDisk -InputObject $diskToAdd
Write-Host "Cluster disk added: $($diskToAdd.Name)"
# Click Storage → Disks to see all available disks. You should see your shared disk listed as Available Storage.

# Configure the cluster quorum settings if needed
# Find cluster disk resource
$clusterDisk = Get-ClusterDisk | Where-Object { $_.Size -lt 6GB }  # pick small quorum disk
Set-ClusterQuorum -NodeAndDiskMajority $clusterDisk

<#
If Get-ClusterDisk does not work, troubleshoot the problem. 
You can use the following as alternative

- Click on your cluster, and in the Actions pane (right side), click More Actions → Configure Cluster Quorum Settings.
- Quorum Configuration Wizard opens. Click Next.
- Choose Select the quorum witness → click Next.
- Select Configure a disk witness → Select the quorum disk.
- Click Next, review settings, then click Finish
- Click on your cluster and review Cluster Core Resources.
#>
```

#### Add the data disk (run on Node01)

- Add the data disk and prepare for SQL FCI (run on Node01)
- Initialize (format) the disk on Node01, add to cluster

```powershell
# Show all disks and find the one that is the iSCSI LUN you want (One of the disks with RAW partition style)
Get-Disk | Sort-Object Size | Format-Table Number, FriendlyName, Size, OperationalStatus, PartitionStyle

# Example: assume data disk is Disk 2 (replace DiskNumber ($dataDiskNumber) accordingly)
$dataDiskNumber = (Get-Disk | Where-Object { $_.Size -ge (31 * 1GB) -and $_.PartitionStyle -eq 'RAW' } | Select-Object -First 1).Number
Write-Host "Data disk number is $dataDiskNumber"

# Bring disk online (if offline), initialize, create partition, format (no drive letter)
# Check the disk state
Get-Disk -Number $dataDiskNumber | Select Number, OperationalStatus, IsOffline, IsReadOnly, PartitionStyle

Set-Disk -Number $dataDiskNumber -IsOffline $false
Set-Disk -Number $dataDiskNumber -IsReadOnly $false

Initialize-Disk -Number $dataDiskNumber -PartitionStyle GPT

$part = New-Partition -DiskNumber $dataDiskNumber -UseMaximumSize -AssignDriveLetter
Format-Volume -Partition $part -FileSystem NTFS -NewFileSystemLabel "SQLSharedData01" -Confirm:$false

# Add the data disk to the Windows Failover Cluster

# Note the disk number 
Get-ClusterAvailableDisk | Format-Table -AutoSize

# Add the disk to cluster
$diskToAdd = Get-ClusterAvailableDisk | Where-Object { ($_.Number -eq 2) }

Add-ClusterDisk -InputObject $diskToAdd
Write-Host "Cluster disk added: $($diskToAdd.Name)"
# Click Storage → Disks to see all available disks. You should see your shared disk listed as Available Storage.
```

#### Install SQL Server

You can create a scenario like below

- Install SQL Server Failover Cluster on Node01 and Node02
  - Failover Group: SQL Workload 01
  - Network Name: SQLCluster01
  - Network: 10.2.100.71
  - Instance: MSSQLSERVER
  - Service: Domain user account
  - SQL Server Version: SQL Server 2019
- Install SQL Server Standalone on Node03
  - Instance 1: INST01
  - Instance 2: INST02
  - Service: Domain user account
  - SQL Server Version: SQL Server 2019
- Install SQL Server Standalone on Node04
  - Instance 1: MSSQLSERVER
  - Instance 2: INST01
  - Instance 3: INST02
  - Service: Domain user account
  - SQL Server Version: SQL Server 2014 on MSSQLSERVER
  - SQL Server Version: SQL Server 2016 on INST01
  - SQL Server Version: SQL Server 2019 on INST02
- Enable firewall for SQL ports in each node


    ```powershell
    # Node01

    New-NetFirewallRule -DisplayName "SQL_TCP_1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_UDP_1434" -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_5022" -Direction Inbound -Protocol TCP -LocalPort 5022 -Action Allow -Profile Any

    # Node02

    New-NetFirewallRule -DisplayName "SQL_TCP_1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_UDP_1434" -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_5022" -Direction Inbound -Protocol TCP -LocalPort 5022 -Action Allow -Profile Any

    # Node03

    New-NetFirewallRule -DisplayName "SQL_TCP_49986" -Direction Inbound -Protocol TCP -LocalPort 49986 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_50123" -Direction Inbound -Protocol TCP -LocalPort 50123 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_UDP_1434" -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_5022" -Direction Inbound -Protocol TCP -LocalPort 5022 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_5023" -Direction Inbound -Protocol TCP -LocalPort 5023 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_5022_Outbound" -Direction Outbound -Protocol TCP -LocalPort 5022 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_5023_Outbound" -Direction Outbound -Protocol TCP -LocalPort 5023 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "MI-Link-Outbound-11000-11999" `
    -Direction Outbound `
    -LocalPort 11000-11999 `
    -Protocol TCP `
    -Action Allow `
    -Profile Any
    New-NetFirewallRule -DisplayName "MI-Link-Inbound-11000-11999" `
    -Direction Inbound `
    -LocalPort 11000-11999 `
    -Protocol TCP `
    -Action Allow `
    -Profile Any


    # Node04

    New-NetFirewallRule -DisplayName "SQL_TCP_1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_UDP_1434" -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_56839" -Direction Inbound -Protocol TCP -LocalPort 56839 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_56821" -Direction Inbound -Protocol TCP -LocalPort 56821 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_5022" -Direction Inbound -Protocol TCP -LocalPort 5022 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_TCP_5022_Outbound" -Direction Outbound -Protocol TCP -LocalPort 5022 -Action Allow -Profile Any    
    New-NetFirewallRule -DisplayName "SQL_TCP_5023" -Direction Inbound -Protocol TCP -LocalPort 5023 -Action Allow 
    New-NetFirewallRule -DisplayName "SQL_TCP_5023_Outbound" -Direction Outbound -Protocol TCP -LocalPort 5023 -Action Allow -Profile Any 
    New-NetFirewallRule -DisplayName "MI-Link-Outbound-11000-11999" `
    -Direction Outbound `
    -LocalPort 11000-11999 `
    -Protocol TCP `
    -Action Allow `
    -Profile Any
    New-NetFirewallRule -DisplayName "MI-Link-Inbound-11000-11999" `
    -Direction Inbound `
    -LocalPort 11000-11999 `
    -Protocol TCP `
    -Action Allow `
    -Profile Any    
    ```
- Test port connectivity (SQL Server Port and SQL Server Browser Port)
  - Test-NetConnection `<SQLServerName>` -Port `<SQLServerPort>`
  - Test-NetConnection `<SQLServerName>` -Port 1434

#### Create databases on SQL Server
- Install Wide World Importers database. [Learn more.](https://github.com/Microsoft/sql-server-samples/releases/tag/wide-world-importers-v1.0)
- Install AdventureWorks database. [Learn more.](https://github.com/Microsoft/sql-server-samples/releases/tag/adventureworks)
- Install Northwind and Pubs databases. [Learn more.](https://github.com/microsoft/sql-server-samples/tree/master/samples/databases/northwind-pubs)

#### Create availability group and lister
- Availability Group
  - In the scenario, we setup availability group between **SQLCluster01**, which is SQL Server Failover Cluster on Node01 and Node02, and **Node03\INST01**
  - Enable Availability group through SQL Server Consfiguration Manager
    - Enable it on Node01 and failover it to Node02, which will enable it on Node02 as well because it is the same SQL instance.
    - Enable it on Node03\INST01
  - Backup one of your databases, which is on FULL recovery mode, on SQLCluster01 and restore it on Node03\INST01 with **No Recovery** option.
  - Create availability group between these 2 instances and select **Join** only option since the database is restored on Node03\INST01.
  - Create **listener**
    - DNS Name: Forexample: SQLListener01
    - Port: 1433 (The AG Listener port is not the same as the SQL Instance’s port binding. The listener port is a virtual endpoint. Clients connect to SQLListener01:1433 and SQL Server routes the connection to SQLCluster01:1433 (your FCI primary) or Node03\INST01:<dynamic_port> (your standalone instance))

## Fix

With this setup, Nested VMs can reach to outside like Test-NetConnection, ping but It does not accept any connection from outside. Only Nested VMs can reach to each other.

So we will change it to external switch

```powershell
# List adapters
Get-NetAdapter | ft Name, Status, MacAddress, LinkSpeed

# Replace "Ethernet" below with the adapter name that is connected to vnet-onprem
New-VMSwitch -Name "vSwitch-External" `
             -NetAdapterName "Ethernet" `
             -AllowManagementOS $true `
             -Notes "External switch for nested VMs on 10.2.0.0/16"

# List adapters
Get-NetAdapter | ft Name, Status, MacAddress, LinkSpeed

# Detach & attach NIC to External switch
$vm = "Node04"

# Disconnect NIC from NAT switch
Get-VMNetworkAdapter -VMName $vm | Disconnect-VMNetworkAdapter -Passthru

# Connect NIC to External vSwitch
Get-VMNetworkAdapter -VMName $vm | Connect-VMNetworkAdapter -SwitchName "vSwitch-External" -Passthru



# Verify IP config (power on if needed) inside the VM
# From Hyper-V host, using PowerShell Direct
Invoke-Command -VMName $vm -ScriptBlock {
  Write-Host "Current IP config:"
  Get-NetIPConfiguration | Select-Object InterfaceAlias, IPv4Address, IPv4DefaultGateway, DNSServer | Format-List

  # If adapter is present but no IP (rare), reapply the static IP that the VM had before:
  # Example: (DO NOT run unless needed; adjust IPs)
  # New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.2.100.15 -PrefixLength 24 -DefaultGateway 10.2.0.1
  # Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("10.2.100.11")
}

Important: Use the same IP that the VM had before (10.2.100.x). This keeps cluster IPs and listener IPs valid.

```