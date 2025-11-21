# Create Oracle Environment

This environment provisions an Oracle XE 21c instance on an Oracle Linux 8.10 virtual machine, deployed within a workload VNet (spoke) as part of a hub-and-spoke topology. The VM serves as the source system for schema and data migration to Azure Database for PostgreSQL Flexible Server.

In this repository, the Oracle VM is primarily used to:

- Host and manage Oracle XE 21c databases for migration testing
- Enable connectivity from a jumpbox VM for secure extraction and conversion
- Facilitate schema conversion and data movement to PostgreSQL using Azure tooling

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
$resourceGroup="rg-oracle01"
$vnet="vnet-oracle01"
$backendSubnet="subnet-vnet-oracle01-backend"
$backendSubnetNSG="nsg-subnet-vnet-oracle01-backend"
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$jumpboxResourceGroup="rg-jumpbox"
$jumpboxVNet="vnet-jumpbox"
$networkInterface = "nic-OracleVM01"
$SSH_KEY_NAME="oraclevm01-key-ed"
$vmName="OracleVM01"
$osDisk="disk1-OracleVM01"
$rdpRuleName01 = "AllowSSH"
$sourceIP01 = "<your source ip>/32"   # Private subnet range of JumpboxVM01 - 10.1.0.0/24
$priority01 = 100  # Lower number = higher priority
$rdpRuleName02 = "AllowConnectionToOracle"
$sourceIP02 = "<your source ip>/32"   # Private subnet range of JumpboxVM01 - 10.1.0.0/24
$priority02 = 110  # Lower number = higher priority
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create resource group for the Oracle

```powershell
# Create a resource group
az group create `
  --name $resourceGroup `
  --location $location
```

### Step 4: Create the Oracle virtual network and subnet

```powershell
# Create a virtual network with a subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.5.0.0/16 `
  --subnet-name $backendSubnet `
  --subnet-prefix 10.5.0.0/24
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
# Peer oracle to hub
az network vnet peering create `
  --name oracle01-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

# Peer hub to oracle
az network vnet peering create `
  --name hub-to-oracle01 `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  

# Peer oracle to jumpbox
az network vnet peering create `
  --name oracle01-to-jumpbox `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$jumpboxResourceGroup/providers/Microsoft.Network/virtualNetworks/$jumpboxVNet" `
  --allow-vnet-access

# Peer jumpbox to oracle
az network vnet peering create `
  --name jumpbox-to-oracle01 `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  
```

### Step 7: Create a network interface to be used by Oracle Virtual Machine

```powershell
# Create network interface with private IP
az network nic create `
  --resource-group $resourceGroup `
  --name $networkInterface `
  --vnet-name $vnet `
  --subnet $backendSubnet `
  --network-security-group $backendSubnetNSG `
  --location $location
```

### Step 8: Create the Oracle VM

#### Generate an SSH key pair to securely connect to the VM, then provision the instance

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

#### Explore Oracle Linux Images

You can list all available Oracle Linux images in Azure before creating your VM.

```powershell
# List available Oracle Linux images (all versions)
az vm image list --publisher oracle --offer Oracle-Linux --output table --all

# Or, list all images from Oracle publisher
az vm image list --publisher Oracle --output table --all
```

#### Deploy Oracle VM (Oracle-Linux 8.10)

⚠️ **Note:** Replace placeholder values like `<your-admin-username>` with your own before running the commands.

```powershell
# Create the Oracle Linux virtual machine
az vm create `
  --resource-group $resourceGroup `
  --name $vmName `
  --location $location `
  --nics $networkInterface `
  --image "Oracle:Oracle-Linux:ol810-lvm-gen2:8.10.7" `
  --size "Standard_D4s_v3" `
  --admin-username "<your-admin-username>" `
  --ssh-key-values "$env:USERPROFILE\.ssh\$SSH_KEY_NAME.pub" `
  --authentication-type ssh `
  --security-type "TrustedLaunch" `
  --enable-agent true `
  --os-disk-size-gb 64 `
  --os-disk-name $osDisk `
  --storage-sku Premium_LRS
```  

### Step 9: Allow SSH Access to the Oracle Linux VM

```powershell
# Create NSG rule to allow SSH
az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $backendSubnetNSG `
  --name $rdpRuleName01 `
  --priority $priority01 `
  --protocol Tcp `
  --source-address-prefixes $sourceIP01 `
  --destination-port-ranges 22 `
  --description "Allow SSH access to Oracle VM from trusted IP"
```

### Step 10: Allow Oracle Database access

```powershell
# Create NSG rule to allow connection to Oracle Server
az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $backendSubnetNSG `
  --name $rdpRuleName02 `
  --priority $priority02 `
  --protocol Tcp `
  --source-address-prefixes $sourceIP02 `
  --destination-port-ranges 1521 `
  --description "Allow Port 1521 access to Oracle VM from trusted IP"
```

### Step 11: Connect to Oracle VM

This guide explains how to connect to your Oracle Linux virtual machine from your laptop or from the JumpboxVM01 using SSH (Secure Shell).

The connection requires using the private key you generated earlier when creating your Oracle VM.

💡 Note:
Oracle is not pre-installed on the VM. You’ll use SSH to log into the Linux OS and install Oracle manually.
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
| `oraclevm01-key-ed`     | Private key - used to authenticate to your VM     |
| `oraclevm01-key-ed.pub` | Public key - uploaded to Azure during VM creation |

#### Copy the private key to Jumpbox01VM and secure the file

RDP to Jumpbox01VM

```powershell
# Create the .ssh folder if it doesn't exist
New-Item -ItemType Directory -Path "$env:USERPROFILE\.ssh" -Force
```

Copy the oraclevm01-key-ed file from your machine to Jumpbox01VM "$env:USERPROFILE\.ssh" folder.

Secure the Key File in Jumpbox01VM: You must set the correct permissions on the key to ensure SSH will accept it.

```powershell
# Run in PowerShell (Admin)
icacls "$env:USERPROFILE\.ssh\oraclevm01-key-ed" /inheritance:r /grant:r "${env:USERNAME}:R"
```

#### Connect to the Oracle VM from Jumpbox01

Open Windows Powershell with Admin and run the following command

```powershell
#Use the following command to connect from your laptop or Jumpbox:
ssh -i $env:USERPROFILE\.ssh\oraclevm01-key-ed azadmin@<oracle-vm-private-ip>
```

💡 Note:
Replace `<oracle-vm-private-ip>` with your actual Oracle VM's private IP address

If you receive the following message, type yes then press enter. SSH will remember this host in your known_hosts file for future sessions.

```text
The authenticity of host '10.5.0.4 (10.5.0.4)' can't be established.
ED25519 key fingerprint is SHA256:GAqQ5DbLkkPox/5iNSwcG/vPCZxBcqgtAuOtfMEawvI.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

#### Troubleshooting SSH Connection

If you see connection timed out, It means your VM's port 22 (SSH) is blocked. Go to Azure Portal → VM → Networking → Inbound Port Rules and add:

| Setting              | Value                              |
| -------------------- | ---------------------------------- |
| **Source**           | "<your source ip>/32"              |
| **Destination Port** | 22                                 |
| **Protocol**         | TCP                                |
| **Priority**         | 1000 (or lower than any Deny rule) |
| **Name**             | AllowSSH                           |

### Step 12: Oracle Database 21c XE 

If you are not connected Oracle-Linux server, run the command below from JumpboxVM01.

```powershell
#Use the following command to connect from your laptop or Jumpbox:
ssh -i $env:USERPROFILE\.ssh\oraclevm01-key-ed azadmin@<oracle-vm-private-ip>
```

Update packages and install dependencies required for Oracle XE.

💡 Note:
- This installs all required dependencies and configures kernel parameters for Oracle XE.
- sudo stands for "superuser do" which allows a regular user to run commands with administrator (root) privileges
- apt stands for Advanced Package Tool which is the package manager for Debian-based systems like Ubuntu
- dnf stands for Dandified Yu which is the package manager for Oracle Linux, RHEL, CentOS, Fedora4

```powershell
# Update all system packages
sudo dnf update -y

# Install Oracle EPEL (Extra Packages for Enterprise Linux)
sudo dnf install -y oracle-epel-release-el8

# Install pre-requisites and common utilities
sudo dnf install -y oracle-database-preinstall-21c wget zip unzip vim
```

Download the RPM package for Oracle Database 21c XE.

```powershell
wget https://download.oracle.com/otn-pub/otn_software/db-express/oracle-database-xe-21c-1.0-1.ol8.x86_64.rpm
```

Install Oracle XE.

```powershell
sudo dnf localinstall -y oracle-database-xe-21c-1.0-1.ol8.x86_64.rpm
```

Run the configuration script to set up the database.

```powershell
sudo /etc/init.d/oracle-xe-21c configure
```

💡 Note:
- Specify a password to be used for database accounts. Oracle recommends that the password entered should be at least 8 characters in length, contain at least 1 uppercase character, 1 lower case character and 1 digit [0-9]. Note that the same password will be used for SYS, SYSTEM and PDBADMIN accounts.

  | Prompt            | Example / Default | Description                                  |
  | ----------------- | ----------------- | -------------------------------------------- |
  | **Listener Port** | `1521`            | Oracle network port                          |
  | **APEX Port**     | `5500`            | For Oracle Application Express (web console) |
  | **Passwords**     | `<password_you_specified>`    | For SYS, SYSTEM, and PDBADMIN                |
  | **Start on Boot** | `Yes`             | Auto-start database on VM boot               |

    After successful configuration, you’ll see the **output**:

    ```text
    Connect to Oracle Database using one of the connect strings:
        Pluggable database: OracleVM01/XEPDB1
        Multitenant container database: OracleVM01
    Use https://localhost:5500/em to access Oracle Enterprise Manager for Oracle Database XE

### Step 13: Post-Installation Configuration

#### Fix permissions, configure the listener, and set Oracle environment variables.

```powershell
# Fix listener log directory permissions
sudo mkdir -p /opt/oracle/product/21c/dbhomeXE/network/log
sudo chown -R oracle:oinstall /opt/oracle/product/21c/dbhomeXE/network/log
```

#### Create a system-wide profile script under /etc/profile.d to make Oracle environment variables available to all users and sessions, 

```powershell
# Create a System-Wide Profile Script
sudo tee /etc/profile.d/oracle_env.sh > /dev/null <<EOF
    export ORACLE_HOME=/opt/oracle/product/21c/dbhomeXE
    export PATH=\$ORACLE_HOME/bin:\$PATH
    export LD_LIBRARY_PATH=\$ORACLE_HOME/lib
    export ORACLE_SID=XE
    export TNS_ADMIN=\$ORACLE_HOME/network/admin
EOF
sudo chmod +x /etc/profile.d/oracle_env.sh

# Execute the script for immediate action
source /etc/profile.d/oracle_env.sh
```

#### Review the listener status and start if not started

```powershell
# Check listener status
lsnrctl status
```

💡 Note: What to Look For:
  - Listener is running
  - Services like XE, XEPDB1, and XEXDB show status READY
  - Listening endpoints include PORT=1521

```powershell
# If Listener Is Not Running, start listener
lsnrctl start
```

#### Test SQL*Plus Connection

```powershell
# Connect as system user
sqlplus system@localhost/XEPDB1

#Exit
Exit
```

### Step 14: Install Git to download sample schemas

As the `<your-admin-username>` user:

```powershell
sudo dnf install -y git
git --version
```

### Step 15: Download Oracle Sample Databases

Clone the official Oracle sample schema repository:

```powershell
cd ~
git clone https://github.com/oracle-samples/db-sample-schemas.git
cd db-sample-schemas
ls -l
```

💡 Note: You should see the following databases. For more information, review the [document](https://github.com/oracle-samples/db-sample-schemas).
- CO (Customer Orders) - OLTP 
- HR (Human Resources) - OLTP
- SH (Sales History) - OLAP

### Step 16: Load Sample Databases - customer_orders

```powershell
cd ~/db-sample-schemas/customer_orders
sqlplus system@localhost/XEPDB1 (or alternate sqlplus system@"localhost:1521/XEPDB1")
@co_install.sql
```
💡 Note: 

- Enter password: `<set password for CO>`
- Tablespace: `Press Enter (defaults to USERS)`
- Do you want to overwrite the schema, if it already exists? [YES|no]: `YES`

Verify data

```powershell
sqlplus co@localhost/XEPDB1
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;
Exit
```

### Step 17: Load Sample Databases - sales_history

```powershell
cd ~/db-sample-schemas/sales_history
sqlplus system@localhost/XEPDB1 (or alternate sqlplus system@"localhost:1521/XEPDB1")
@sh_install.sql
```
💡 Note: 

- Enter password: `<set password for SH>`
- Tablespace: `Press Enter (defaults to USERS)`
- Do you want to overwrite the schema, if it already exists? [YES|no]: `YES`

Verify data

```powershell
sqlplus sh@localhost/XEPDB1
SELECT COUNT(*) FROM sales;
Exit
```

### Step 18: Load Sample Databases - human_resources

```powershell
cd ~/db-sample-schemas/human_resources
sqlplus system@localhost/XEPDB1 (or alternate sqlplus system@"localhost:1521/XEPDB1")
@hr_install.sql
```
💡 Note: 

- Enter password: `<set password for HR>`
- Tablespace: `Press Enter (defaults to USERS)`
- Do you want to overwrite the schema, if it already exists? [YES|no]: `YES`

Verify data

```powershell
sqlplus hr@localhost/XEPDB1
SELECT COUNT(*) FROM employees;
Exit
```

### Step 19: Create a Custom Test User and Table

Connect as SYSTEM

```powershell
sqlplus system@localhost/XEPDB1 (or alternate sqlplus system@"localhost:1521/XEPDB1")
```

Create a new user and grant permissions:

```powershell
CREATE USER test_user IDENTIFIED BY <password>;
GRANT CONNECT, RESOURCE TO test_user;
ALTER USER test_user QUOTA UNLIMITED ON USERS;
Exit
```

Connect as new user

```powershell
sqlplus test_user@localhost/XEPDB1 (or alternate sqlplus test_user@"localhost:1521/XEPDB1")
```

Create and load a sample table

```sql
CREATE TABLE employees (
id NUMBER PRIMARY KEY,
name VARCHAR2(100),
department VARCHAR2(50),
salary NUMBER
);

INSERT INTO employees VALUES (1, 'Alice', 'Engineering', 90000);
INSERT INTO employees VALUES (2, 'Bob', 'Marketing', 75000);
INSERT INTO employees VALUES (3, 'Charlie', 'HR', 60000);

COMMIT;

SELECT * FROM employees;
```

### Step 20: Run queries for the databases

```powershell
# connect customer_orders database
sqlplus co@localhost/XEPDB1 (or alternate sqlplus co@"localhost:1521/XEPDB1")
```

```sql
#Which schema
SHOW USER;

--Which tables
SELECT table_name FROM user_tables ORDER BY table_name;


--List tables by schema
COLUMN owner FORMAT A10
COLUMN table_name FORMAT A30

SELECT owner, table_name
FROM all_tables
WHERE owner = 'CO'
ORDER BY table_name;    

/*
Explanation:
- FORMAT A10 → sets OWNER column to 10 characters wide
- FORMAT A30 → sets TABLE_NAME column to 30 characters wide
This will print the results in a cleaner, single-line format like:
*/

--Select from Orders table
SELECT COUNT(*) FROM orders;

--Exit
exit
```

Run the similar queries for human_resources and sales_history

- sqlplus hr@localhost/XEPDB1 (or alternate sqlplus co@"localhost:1521/XEPDB1"    )
- sqlplus sh@localhost/XEPDB1 (or alternate sqlplus co@"localhost:1521/XEPDB1")

### Step 21: Update listener to be accessible from outside (JumpboxVM01)

SSH to OracleVM01 from JumpboxVM01 with Windows Powershell

```powershell
#Use the following command to connect from your laptop or Jumpbox:
ssh -i $env:USERPROFILE\.ssh\oraclevm01-key-ed azadmin@<oracle-vm-private-ip>
```

Edit listener.ora

```powershell
sudo nano /opt/oracle/homes/OraDBHome21cXE/network/admin/listener.ora
```

Output
```text
DEFAULT_SERVICE_LISTENER = XE

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = oraclevm01.internal.cloudapp.net)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
```

Listener accepts TCP connection from local host. When you SSH to linux server, you can connect to database because host is set to local as **HOST = oraclevm01.internal.cloudapp.net**. 

Update Host to 0.0.0.0 for now to accept connection from everywhere like **HOST = 0.0.0.0**

Save the file (Ctrl O - Press Enter - Ctrl X)

Updated version of listener.ora

```text
DEFAULT_SERVICE_LISTENER = XE

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
```

Switch to oracle user

```powershell
sudo su - oracle
```

Stop and Start listener

```powershell
# Stop listener
lsnrctl stop

# Start listener
lsnrctl start

# Status
lsnrctl status
```

### Step 21: Open port 1521 in OracleVM01 Firewall

SSH to OracleVM01 from JumpboxVM01 with Windows Powershell

```powershell
#Use the following command to connect from your laptop or Jumpbox:
ssh -i $env:USERPROFILE\.ssh\oraclevm01-key-ed azadmin@<oracle-vm-private-ip>
```

Enable port 1521 in OracleVM01 Firewall

```powershell
# You should be connected with azadmin not oracle
sudo firewall-cmd --permanent --add-port=1521/tcp
sudo firewall-cmd --reload
```

### Step 22: Troubleshoot Oracle database connection problems


#### Schema discovery failed: Oracle schema retrieval failed: Failed to connect to Oracle database: DPY-6005: cannot connect to database (CONNECTION_ID=0LZ0Bic9hQViw9N1wPrTZw==). DPY-6001: Service "XEPDB1" is not registered with the listener at host "10.5.0.4" port 1521. (Similar to ORA-12514)

```powershell
Review the listener status
lsnrctl status
```

Listener status should have Service "XEPDB1"

```text
Services Summary...
Service "XEPDB1" has 1 instance(s).
```

If you see a message **The listener supports no services**, which means it’s not aware of any database services, including XEPDB1 or even XE.

**Solution**

SSH to Oracle Database

```powershell
#Use the following command to connect from your laptop or Jumpbox:
ssh -i $env:USERPROFILE\.ssh\oraclevm01-key-ed azadmin@<oracle-vm-private-ip>
```
Connect to the database as SYSDBA

```powershell
sqlplus / as sysdba
```

If you get a message like "invalid username/password; logon denied",  Oracle XE instance is not allowing OS authentication for **sqlplus / as sysdba**.

Check Group Membership

```powershell
groups azadmin
```

You should see:

```text
azadmin : azadmin dba
```

If dba is missing, add it.

```powershell
sudo usermod -aG dba azadmin
```

Then log out and log back in to apply the group change. You can verify with:

```powershell
groups azadmin
```

Expected output (you should see dba):

```text
[azadmin@OracleVM01 ~]$ groups azadmin
azadmin : azadmin adm systemd-journal dba
```

Then try

```powershell
sqlplus / as sysdba
```

Start the Database

```sql
STARTUP;
```

If the database is already started, you’ll see:

```text
ORA-01081: cannot start already-running ORACLE
```

Open the Pluggable Database (XEPDB1)

```sql
-- If you're using Oracle XE 21c with a pluggable architecture:
ALTER PLUGGABLE DATABASE XEPDB1 OPEN;
```

Check Registered Services

```sql
SELECT name FROM v$services;
```

You should see something like:

```text
XE
XEPDB1
```


💡 Note:
You now have a fully configured Oracle XE environment with sample and custom databases.
This setup can be used for migration testing, training, or integration validation with Azure PostgreSQL.