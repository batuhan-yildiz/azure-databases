# Data migration with Azure Data Factory (ADF)

Oracle to Azure Database for PostgreSQL migration

All required resources have been created in .\setup\create-migration-environment.md

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
$resourceGroup="rg-migration"
$vnet="vnet-migration"
$backendSubnet="subnet-vnet-migration-backend"
$backendSubnetNSG="nsg-subnet-vnet-migration-backend"
$storageAccount="samodernization01"
$adfName="adf-modernization01"
$kvName="kv-modernization01"
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Store secrets in key valut

```powershell
# Store Oracle and Azure Database for PostgreSQL passwords in keyvault.
az keyvault secret set --vault-name $kvName --name "oracle-migrationpassword" --value "ORACLE_PASSWORD"
az keyvault secret set --vault-name $kvName --name "pg-migrationpassword" --value "PG_PASSWORD"
```

### Step 4: Install Self-Hosted Integration Runtime (SHIR) on JumpboxVM01

#### Create a new Integration Runtime

- SHIR allows ADF to communicate with Oracle
- Open a new browser and go to [https://adf.azure.com](https://adf.azure.com).
- Select Entra ID, Subscription and recently created Azure Data Factory (adf-modernization01) and click on **Continue**.
- On the left menu, click on **Manage**
- Click on Integration runtimes
  - Click **+ New**
  - Select **Azure, Self-Hosted** and click on **Continue**.
  - Select **Self-Hosted** and click on **Continue**.
  - Give a name "**adf-integrationruntime01**"
  - Click on **Create**
  - Copy Key1 and Key2 somewhere secure
  - Status will show that it is unavailable because you have not installed the integration runtime yet. 

```powershell
# check provisioning status
az datafactory show --resource-group $resourceGroup --name $adfName --query "provisioningState"
``` 

Possible outputs:
- Succeeded → ready to use
- Creating → still provisioning
- Failed → needs attention (rare)

#### Download and Install Integration Runtime
- Oepn a browser on JumpboxVM01
- Search for Microsoft Integration Runtime Download
- [Download](https://www.microsoft.com/en-us/download/details.aspx?id=39717) it.
- Install SHIR on JumpboxVM01
- Once the installation is complete, then paste the authentication key, which you copied Key1 and Key2 before, to register. You should see **Integration Runtime (Self-hosted) node has been registered successfully**.
- Click on **Launch Configuration Manager** and review. It hsould be connected.

### Step 5: Verify ADF can access Key Vault secrets

- Go to [https://adf.azure.com](https://adf.azure.com) portal
- Go to Manage → Linked Services → + New
- Select Azure Key Vault
    - Name: linked_kv_modernization
    - Select Subscription, Azure key vault name, Authentication method (system-assigned)
    - Test connection: To linked service
    - Click on Create
- Create another linked service (Manage → Linked Services → + New)
- Select Oracle
    - Name: linked_oracle_source
    - Connect via integration runtime: Select adf-integrationruntime01
    - Server / TNS: 10.4.0.4:1521/XEPDB1
    - Authentication type: Basic
    - User name: MIGRATIONUSER
    - For password, select Azure Key Vault
      - AKV linked service: linked_kv_modernization
      - Secret name: oracle-migrationpassword
      - Secret version: Latest
      - Click on Test Connection
      - Click on Create
- Create another linked service (Manage → Linked Services → + New)
- Select Azure Database for PostgreSQL
    - Name: linked_postgresql_target
    - Connect via integration runtime: Select adf-integrationruntime01
    - Authentication type: Basic
    - Account selection method: From Azure subscription
      - Select subscription, server name, port and database name
    - User name: azpgsqladmin
    - For password, select Azure Key Vault
      - AKV linked service: linked_kv_modernization
      - Secret name: pg-migrationpassword
      - Secret version: Latest
      - Click on Test Connection
      - Click on Create
- Click Publish all
  - You created linked services and IR in Manage
  - These changes are not live in the “pipeline editor” until you publish.


### Step 6: Move data

In Azure Data Factory, the typical approach is Copy Data activity in a pipeline:

- Source → Oracle linked service
- Sink → PostgreSQL linked service
- You can do:
    - Full table copy (initial load)
    - Incremental load (if you have a timestamp / column to track changes)

Create a Data Set
- Go to Author through adf portal
- Create a new Dataset for source
    - Select Oracle and click on Continue
    - Set name to dataset_oracle_source
    - Select linked_oracle_source which you created and tested the connection through Manage
    - Pick one table for testing and click OK. You can update later
- Create a new Dataset for target
    - Select Azure Database for PostgreSQL and click on Continue
    - Set name to dataset_oracle_source
    - Select linked_postgresql_target which you created and tested the connection through Manage
    - Pick the right table which you picked from source and click OK. You can update later
    - Import schema: None since we migrated schema before

Add copy data activity
- Open your pipeline
- Drag Copy Data activity to the canvas
- Source tab → Dataset: select dataset_oracle_source
- Sink tab → 
  - Dataset: select dataset_pgsql_target
  - Select Upsert (update and insert)
- Mapping tab: 
  - Clcik on import schema since the schema has already been migrated
  - It will automatically map the columns between source and target
- Click on Debug
  - Execute the pipeline on-demand using your SHIR. This is your test run. It does not require Publish, but lets you:
    - Check that the pipeline actually copies data
    - See rows copied
    - Catch any errors (data type, constraints, etc.)
- Click on Validate
  - While still in pipeline canvas, click Validate all (top menu). This checks for any errors in your pipeline:
    - Dataset issues
    - Missing parameters
    - Mapping issues
- Publish all
  - Click Publish All (top menu, the number indicates pending changes). This saves your pipeline, datasets, and mappings to the live ADF environment. Required before running in production or scheduling triggers.
- Production run
  - After successful debug and publish, run the pipeline for production:
    - Option 1: Click Trigger → Trigger Now → executes pipeline immediately in live environment
    - Option 2: Schedule trigger if incremental load is needed.
- Monitor & validate
  - Go to Monitor tab → Pipeline runs:
    - Ensure status = Succeeded
    - Check row counts in PostgreSQL match Oracle
    - Sample data to validate correctness

Review data at your target
- Connect to Azure Database for Postgresql through JumpboxVM01 VS Code
- Execute select statement againt the table you migrated