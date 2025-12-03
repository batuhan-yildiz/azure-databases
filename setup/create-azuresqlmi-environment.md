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
$onpremResourceGroup="rg-onprem"
$onpremVNet="vnet-onprem"
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
<#
💡Note: There are Nested VMs on HyperHostVM01. 
   We will test Arc-enabled SQL Server migration to Azure SQL Managed instance.
   Migration can bu run through JumpboxVM01 through azure portal, but JumpboxVM01 plays zero network role except running the UI.
   If we run a migration from nested VM to Azure SQL MI, these 2 networks must have been peered. 
   Traffic cannot follow Nesterd VM → OnPremVNet → JumpboxVNet → SQLMIVNet
   You must have direct peering between OnPremVNet and SQLMIVNet
   When there is a gateway or nested routing scenario between the networks, Allow forwarded traffic must be enabled.
   In our migration scenaris, the migration will be from nested VM to SQL MI. So, allow-forwarded-traffic must be enabled between OnPremVNet and SQLMIVNet.
   So, the following 2 peerings must be created.
#>

# Peer sqlmi01 to onprem
az network vnet peering create `
  --name sqlmi01-to-onprem `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$onpremResourceGroup/providers/Microsoft.Network/virtualNetworks/$onpremVNet" `
  --allow-vnet-access `
  --allow-forwarded-traffic

# Peer onprem to sqlmi01
az network vnet peering create `
  --name onprem-to-sqlmi01 `
  --resource-group $onpremResourceGroup `
  --vnet-name $onpremVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access `
  --allow-forwarded-traffic
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

### Step 11: Create a VPN Gateway

It will be used for a database migration from a nested VM to Azure SQL Managed Instance. 

```powershell
# variables for gateway
$gwName = "gw-$vnet"
$publicIPGW = "pip-$gwName"
$tempPath = "C:\Temp\P2S"
$rootCertName = "PSRootCert01"
$certPassword = Read-Host -Prompt "Enter certificate password:" -AsSecureString

# Add a GatewaySubnet to vnet-sqlmi01
az network vnet subnet create `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --name "GatewaySubnet" `
  --address-prefixes 10.3.255.0/27

# Create public IP for VPN Gateway
az network public-ip create `
  --resource-group $resourceGroup `
  --name $publicIPGW `
  --sku Standard `
  --allocation-method Static `
  --location $location

# Create VPN Gateway
az network vnet-gateway create `
  --resource-group $resourceGroup `
  --name $gwName `
  --public-ip-address $publicIPGW `
  --vnet $vnet `
  --gateway-type Vpn `
  --vpn-type RouteBased `
  --sku VpnGw1 `
  --location $location 

# VPN Gateway Status
az network vnet-gateway show `
  --resource-group $resourceGroup `
  --name $gwName -o table

# View the public IP address
az network public-ip show `
  --name $publicIPGW `
  --resource-group $resourceGroup

# add the VPN client address pool
az network vnet-gateway update `
  --resource-group $resourceGroup `
  --name $gwName `
  --address-prefixes 172.16.201.0/24 `
  --client-protocol IkeV2 SSTP

# Create a self-signed root certificate
# The root cert public key will be uploaded to Azure. The client cert (issued from this root) will be installed on nested VM.
# Review the certificate by opening certmgr.msc. 
# Certificate will be automatically installed in [Certificates-Current User]\Personal\Certificates.
$rootCert = New-SelfSignedCertificate `
  -Type Custom `
  -KeySpec Signature `
  -Subject "CN=AzureP2SRootCert" `
  -KeyExportPolicy Exportable `
  -HashAlgorithm sha256 `
  -KeyLength 2048 `
  -CertStoreLocation "Cert:\CurrentUser\My" `
  -FriendlyName "AzureP2SRootCert" `
  -KeyUsageProperty Sign `
  -KeyUsage CertSign

# Generate a client certificate
# Each client computer that connects to a VNet using Point-to-Site must have a client certificate installed. # You generate a client certificate from the self-signed root certificate, and then export and install the client certificate

$clientCert = New-SelfSignedCertificate `
  -Type Custom `
  -DnsName AzureP2SChildCert `
  -KeySpec Signature `
  -Subject "CN=AzureP2SChildCert" `
  -KeyExportPolicy Exportable `
  -HashAlgorithm sha256 `
  -KeyLength 2048 `
  -CertStoreLocation "Cert:\CurrentUser\My" `
  -FriendlyName "AzureP2SChildCert" `
  -Signer $rootCert `
  -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2")

# Find the Thumb Print of Certificate
Get-ChildItem Cert:\CurrentUser\My | Where-Object { $_.Subject -like "*AzureP2S*" } | Select-Object Subject, Thumbprint

# Export the root certificate public key (.cer)
$rootCert1 = (Get-ChildItem -Path cert:\CurrentUser\My\06FBC618F9DD65ED93BD72CD781BA5CAFD511F6C)
Export-Certificate -Cert $rootCert1 -FilePath "c:\temp\p2s\AzureP2SRootCert.cer"

# The above Export-Certificate cmdlet does not export the "Base-64 encoded X.509 (.CER)" type
# Run the following command to convert the type
certutil.exe -encode "c:\temp\p2s\AzureP2SRootCert.cer" "c:\temp\p2s\AzureP2SRootCertBase64.cer"

# Upload the root certificate public key information
# Upload the .cer file (which contains the public key information) for a trusted root certificate to Azure
# You created the root certificate like -Subject "CN=AzureP2SRootCert" `
# Name will be AzureP2SRootCert
az network vnet-gateway root-cert create `
--resource-group $resourceGroup `
--name "AzureP2SRootCert" `
--gateway-name $gwName `
--public-cert-data "c:\temp\p2s\AzureP2SRootCertBase64.cer"

# Find the Thumb Print of Certificate for Client
Get-ChildItem Cert:\CurrentUser\My | Where-Object { $_.Subject -like "*AzureP2S*" } | Select-Object Subject, Thumbprint

# Export client certificate in .pfx file with a strong password
Get-ChildItem -Path cert:\CurrentUser\My\10CE11F63C0279FC7A1065708321917DED8E67A6 `
| Export-PfxCertificate -FilePath "c:\temp\p2s\ClientCertificate.pfx" -Password $certPassword


# Copy the file ClientCertificate.pfx to Node04

# Double-click on ClientCertificate.pfx in Node04 
# Under Certificate Import Wizard, select Current User
# Select the file to import and click on Next
# Type the Password and click on Next
# Store the certificate under Current User → Personal and click on Next
# Click on Finish
# Be sure to get **The import was successful** message

# The command outputs a URL to a zip file for the generated VPN client configuration.
# Command will generate a URL to download the vpn client zip file
az network vnet-gateway vpn-client generate `
--resource-group $resourceGroup `
--name $gwName `
--processor-architecture Amd64

# Copy the zip file to Node04
# Extract the zip file in Node04
# Install the VPN Client
# Run the VPN Client to connect

```
