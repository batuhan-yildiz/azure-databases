# Create Windows and SQL Arc Environment

The HyperV server is a virtual machine deployed in a workload VNet (spoke) that allows secure administrative access to resources in private networks.

In this repository, this VM will primarily be used to access:

- Hyper-V and nested VMs
- Windows and SQL Arc

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
$switchName = "NAT-Switch"
$hostInternalIP = "10.2.100.1"
$hostInternalPrefixLen = 24
$nestedPrefix = "10.2.100.0/24"
$natName = "NestedNAT"
$isoPath = "C:\Products\WindowsServer2022.iso"
$vmStorePath = "E:\HyperV\Virtual Hard Disks"
$vmConfigStorePath = "E:\HyperV\Virtual Machines"

# Nested VMs General
$vmMemoryDC = 4GB
$vmMemoryNode = 6GB
$vmProcessorCount = 2
$vmVhdSizeGB = 80   # size for each nested VM VHDX
$gateway = $hostInternalIP
$dcNetmask = 24
$localAdminUser = "Administrator"  # will be used when automating into guest
$plainPassword = Read-Host -Prompt "Enter nested VMs local Administrator password (will be used for PowerShell Direct)" -AsSecureString
$cred = New-Object System.Management.Automation.PSCredential($localAdminUser, $plainPassword)

# DC01 Nested VM
$dcVmName = "DC01"
$dcIp = "10.2.100.11"
$domainName = "contoso.local"
$domainAdminUserUPN = "Administrator@contoso.local"

# Node01 Nested VM
$nodeVm01 = "Node01"
$nodeIp01 = "10.2.100.12"

# Node02 Nested VM
$nodeVm02 = "Node02"
$nodeIp02 = "10.2.100.13"

# Node03 Nested VM
$nodeVm03 = "Node03"
$nodeIp03 = "10.2.100.14"

# Node04 Nested VM
$nodeVm04 = "Node04"
$nodeIp04 = "10.2.100.15"

# Node05 Nested VM
$nodeVm05 = "Node05"
$nodeIp05 = "10.2.100.16"
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
  --allow-vnet-access

# Peer jumpbox to onprem
az network vnet peering create `
  --name jumpbox-to-onprem `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access
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
New-Item -ItemType Directory -Path $vmStorePath
New-Item -ItemType Directory -Path $vmConfigStorePath
```

#### Install Hyper-V and configure

```powershell
# Install Hyper-V role and restart the VM
Install-WindowsFeature -Name Hyper-V -IncludeAllSubFeature -IncludeManagementTools -Restart:$false

# Enable IP routing in Windows so host can forward traffic
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name "IPEnableRouter" -Value 1

# Create internal Hyper-V switch for nested VMs
if (-not (Get-VMSwitch -Name $switchName -ErrorAction SilentlyContinue)) {
    New-VMSwitch -Name $switchName -SwitchType Internal | Out-Null
} else {
    Write-Host "Switch $switchName already exists"
}

# Assign IP to host vEthernet interface bound to the new switch
$iface = "vEthernet ($switchName)"
Write-Host "Assigning IP $hostInternalIP/$hostInternalPrefixLen to interface $iface..."
# remove any existing duplicate IP on that interface first (safe)
Get-NetIPAddress -InterfaceAlias $iface -ErrorAction SilentlyContinue | Remove-NetIPAddress -Confirm:$false -ErrorAction SilentlyContinue
New-NetIPAddress -InterfaceAlias $iface -IPAddress $hostInternalIP -PrefixLength $hostInternalPrefixLen

# Enable forwarding on that interface
Set-NetIPInterface -InterfaceAlias $iface -Forwarding Enabled

# Create NAT so nested VMs have internet outbound via host
Write-Host "Creating NAT for $nestedPrefix ..."
if (-not (Get-NetNat -Name $natName -ErrorAction SilentlyContinue)) {
    New-NetNat -Name $natName -InternalIPInterfaceAddressPrefix $nestedPrefix | Out-Null
} else {
    Write-Host "NAT $natName already exists"
}

# Make sure Windows Firewall allows forwarding/NAT (optional guidance)
Write-Host "Allowing required firewall rules for NAT/forwarding..."
# This leaves default firewall on; NAT works via kernel NAT. If you have explicit firewall policies, ensure forwarding is allowed.

# Create storage folder for nested VHDs (if needed)
if (-not (Test-Path $vmStorePath)) {
    New-Item -Path $vmStorePath -ItemType Directory | Out-Null
}
```

#### Create DC01

```powershell
# Create DC01 VM (Generation 2) and attach ISO
$dcVhd = Join-Path $vmStorePath "$dcVmName.vhdx"
Write-Host "Creating VHDX for $dcVmName..."
New-VHD -Path $dcVhd -SizeBytes ($vmVhdSizeGB * 1GB) -Dynamic | Out-Null

Write-Host "Creating VM $dcVmName..."
New-VM -Name $dcVmName -MemoryStartupBytes $vmMemoryDC -Generation 2 -VHDPath $dcVhd -SwitchName $switchName | Out-Null
Set-VMProcessor -VMName $dcVmName -Count $vmProcessorCount
Add-VMDvdDrive -VMName $dcVmName -Path $isoPath
Set-VM -Name $dcVmName -AutomaticStartAction StartIfRunning -AutomaticStopAction ShutDown

# Start the VMs so OS installation begins (you will need to go into each VM console/Connect to install OS from ISO)
Write-Host "Starting VMs (use Virtual Machine Connection to complete OS install) ..."
Start-VM -Name $dcVmName
# IMPORTANT: connect to each VM (Hyper-V Manager -> Connect) and complete the Windows installation from the ISO.

# Change the computer name to DC01
Invoke-Command -VMName $dcVmName -Credential $cred -ScriptBlock {
    param($newName)

    Rename-Computer -NewName $newName -Force -Restart
} -ArgumentList $dcVmName

# Configure DC01 network inside guest and install AD DS
Write-Host "Configuring network on $dcVmName ..."
Invoke-Command -VMName $dcVmName -Credential $cred -ScriptBlock {
    param($ip,$prefix,$gateway,$dns)
    # set static IP
    $if = Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.Name -like "*Ethernet*" } | Select-Object -First 1
    if ($null -eq $if) { throw "Cannot find network adapter inside guest." }
    $ifAlias = $if.Name
    # Remove DHCP address if present
    Get-NetIPConfiguration -InterfaceAlias $ifAlias | ForEach-Object {
        $_.IPv4Address | ForEach-Object { Remove-NetIPAddress -InterfaceAlias $ifAlias -Confirm:$false -IPAddress $_.IPAddress -ErrorAction SilentlyContinue }
    }
    New-NetIPAddress -InterfaceAlias $ifAlias -IPAddress $ip -PrefixLength $prefix -DefaultGateway $gateway
    Set-DnsClientServerAddress -InterfaceAlias $ifAlias -ServerAddresses $dns
} -ArgumentList $dcIp, $dcNetmask, $gateway, $dcIp -ErrorAction Stop

# Promote DC01 to Domain Controller (create a new forest)
# There will be auto restart and then login with domain Administrator account
Write-Host "Installing AD DS on $dcVmName and promoting to Domain Controller..."
Invoke-Command -VMName $dcVmName -Credential $cred -ScriptBlock {
    param($domainName, $plainPassword)
    # Install AD DS role
    Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

    # convert password string to secure string
    $securePwd = ConvertTo-SecureString $plainPassword -AsPlainText -Force

    # Promote to DC (creates forest)
    Install-ADDSForest -DomainName $domainName `
        -DomainNetbiosName ($domainName.Split('.')[0]) `
        -SafeModeAdministratorPassword $securePwd `
        -InstallDns `
        -Force
} -ArgumentList $domainName, $plainPassword -ErrorAction Stop
```

#### Connect to DC01 Nested VM and create groups and users

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
    Write-Host "User '$groupName' created."
} else {
        Write-Host "User '$groupName' already exists."
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

#### Create Nodes

```powershell
# Select the right node
<#
$nodeVm = $nodeVm01
$nodeIp = $nodeIp01

$nodeVm = $nodeVm02
$nodeIp = $nodeIp02

$nodeVm = $nodeVm03
$nodeIp = $nodeIp03

$nodeVm = $nodeVm04
$nodeIp = $nodeIp04

$nodeVm = $nodeVm05
$nodeIp = $nodeIp05
#>
```

- Run one of the $nodeVm and $nodeIp above
- Execute the following code for each VM. 

```powershell
$nodeVhd = Join-Path $vmStorePath "$nodeVm.vhdx"
Write-Host "Creating VHDX for $nodeVm..."
New-VHD -Path $nodeVhd -SizeBytes ($vmVhdSizeGB * 1GB) -Dynamic | Out-Null

Write-Host "Creating VM $nodeVm..."
New-VM -Name $nodeVm -MemoryStartupBytes $vmMemoryNode -Generation 2 -VHDPath $nodeVhd -SwitchName $switchName | Out-Null
Set-VMProcessor -VMName $nodeVm -Count $vmProcessorCount
Add-VMDvdDrive -VMName $nodeVm -Path $isoPath
Set-VM -Name $nodeVm -AutomaticStartAction StartIfRunning -AutomaticStopAction ShutDown

# Start the VMs so OS installation begins (you will need to go into each VM console/Connect to install OS from ISO)
Write-Host "Starting VMs (use Virtual Machine Connection to complete OS install) ..."
Start-VM -Name $nodeVm
# IMPORTANT: connect to each VM (Hyper-V Manager -> Connect) and complete the Windows installation from the ISO.

# Change the computer name
Invoke-Command -VMName $nodeVm -Credential $cred -ScriptBlock {
    param($newName)

    Rename-Computer -NewName $newName -Force -Restart
} -ArgumentList $nodeVm

# Configure Node network inside guest to use DC for DNS
Write-Host "Configuring network on $nodeVm ..."
Invoke-Command -VMName $nodeVm -Credential $cred -ScriptBlock {
    param($ip,$prefix,$gateway,$dns)
    $if = Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.Name -like "*Ethernet*" } | Select-Object -First 1
    if ($null -eq $if) { throw "Cannot find network adapter inside guest." }
    $ifAlias = $if.Name
    # Remove DHCP address if present
    Get-NetIPConfiguration -InterfaceAlias $ifAlias | ForEach-Object {
        $_.IPv4Address | ForEach-Object { Remove-NetIPAddress -InterfaceAlias $ifAlias -Confirm:$false -IPAddress $_.IPAddress -ErrorAction SilentlyContinue }
    }
    New-NetIPAddress -InterfaceAlias $ifAlias -IPAddress $ip -PrefixLength $prefix -DefaultGateway $gateway
    Set-DnsClientServerAddress -InterfaceAlias $ifAlias -ServerAddresses $dns
} -ArgumentList $nodeIp, $dcNetmask, $gateway, $dcIp -ErrorAction Stop

# Join Node to domain (example using PowerShell Direct)
# There will be auto restart and then login with domain Administrator account
Write-Host "Joining Node to domain..."
Invoke-Command -VMName $nodeVm -Credential $cred -ScriptBlock {
    param($domainName,$domainAdminUserUPN,$securePwd)
    $domainCred = New-Object System.Management.Automation.PSCredential($domainAdminUserUPN,$securePwd)
    Add-Computer -DomainName $domainName -Credential $domainCred -Restart
} -ArgumentList $domainName, $domainAdminUserUPN, $plainPassword -ErrorAction Stop
```

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
  - Service: Domain user account
  - SQL Server Version: SQL Server 2014
- Enable firewall for SQL ports in each node
<br>
<br>
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

    # Node04

    New-NetFirewallRule -DisplayName "SQL_TCP_1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow -Profile Any
    New-NetFirewallRule -DisplayName "SQL_UDP_1434" -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow -Profile Any    
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