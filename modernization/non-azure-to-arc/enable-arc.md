# Deploy Arc-enabled SQL Server

This document describes how to deploy Arc-enabled SQL Server.

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
$resourceGroup="rg-azurearc"
$spName="spn-arc-onboarding"
$nodes=@("Node01","Node02","Node03","Node04","Node05")
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for the arc

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

💡 Note:
A Resource Group (RG) acts as a logical container for your resources.
The rg-hub group will store all shared network components that multiple workloads will depend on.

### Step 4: Prerequisite – Register resource providers

```powershell
# Registering Azure Arc providers
$providers = @(
    "Microsoft.HybridCompute",
    "Microsoft.GuestConfiguration",
    "Microsoft.HybridConnectivity",
    "Microsoft.AzureArcData",
    "Microsoft.Compute"
)
foreach ($p in $providers) {
    az provider register --namespace $p | Out-Null
}
Write-Host "Providers registration requested." -ForegroundColor Green

# Validate the provider registrations
az provider show --namespace Microsoft.HybridCompute --query "registrationState"
az provider show --namespace Microsoft.GuestConfiguration --query "registrationState"
az provider show --namespace Microsoft.HybridConnectivity --query "registrationState"
az provider show --namespace Microsoft.AzureArcData --query "registrationState"
az provider show --namespace Microsoft.Compute --query "registrationState"
```

### Step 5: Create service principal

Roles for Azure Arc onboarding
- Contributor role
    - Scope: Resource Group (or Subscription)
    - Permissions: Full control over all resources except assigning roles.
    - Pros:
        - Simple — covers all operations on the RG, including creating Arc resources.
        - Works for automation scripts, SPNs, DevOps pipelines.
    - Cons:
        - Over-permissioned if you only need Arc onboarding.
        - Security best practices: not ideal to give full Contributor to service principals or people just onboarding servers.
- Azure Connected Machine Onboarding role
    - Scope: Resource Group (or Subscription)
    - Permissions: Only what’s needed to onboard Arc-enabled servers.
    - Pros:
        - Principle of least privilege — secure.
        - Only allows onboarding operations, not creating other Azure resources.
    - Cons:
        - May need extra roles if your SPN/script also needs to install extensions (like SQL extension) because that sometimes requires more permissions.
        - Less flexible for complex automation or multi-step scripts.

```powershell
# Creating SPN for Arc onboarding
$spJson = az ad sp create-for-rbac `
    --name $spName `
    --role "Azure Connected Machine Onboarding" ` `
    --scopes "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup" `
    --output json

$sp = $spJson | ConvertFrom-Json

$appId  = $sp.appId
$secret = $sp.password
$tenant = $sp.tenant

Write-Host "SPN created:"
Write-Host "  App ID:      $appId"
Write-Host "  Secret:      $secret"
Write-Host "  Tenant ID:   $tenant"
```

### Step 6: Generate Arc deployment script

- Go to rg-azurearc resource group
- Click on **Create**
- Search for **Azure Arc**
- Select **Servers - Azure Arc**
  - Select the right subscription
  - Select **Servers - Azure Arc** plan
  - Click on **Create**

#### Onboard existing machines with Azure Arc

- Basics
  - Select subscription
  - Resource group: rg-azurearc
  - Region: (US) West US 3
  - Operating System: Windows
  - Connect SQL Server: True (Automatically connect any SQL Server instances to Azure Arc. [Learn more](https://aka.ms/OptOutOfSqlAutomaticConnection))
  - Connectivity method: Public endpoint
  - Proxy server URL: Leave empty
  - Arc gateway: Leave empty
  - Authentication: Authenticate machines automatically
    - Select the SPN created recently
  - Click on **Next**
- Tags 
  - Click on **Next**
- Download and Run script
  - Click on **Download**

#### Update the onboarding script

- Open PowerShell ISE
- Open the downloaded script in PowerShell ISE
- Find the line **$ServicePrincipalClientSecret="`<ENTER SECRET HERE>`";** in the script
- Update the secret key with SPN secret.
- Copy the script to each node which you want to deploy Arc and execute inside node