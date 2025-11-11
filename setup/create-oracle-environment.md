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
$subnet="subnet-oracle01"
$subnetSecurity="nsg-subnet-oracle01"
```

💡 Note:
Defining variables at the top helps make your scripts reusable.
You can change the region or naming convention once here, and all following commands will automatically use those values.

### Step 3: Create a Resource Group for the Oracle

```powershell
# Create a resource group
az group create --name $resourceGroup --location $location
```

### Step 4: Create the Oracle Virtual Network and Subnet

```powershell
# Create a virtual network with a subnet
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnet `
  --address-prefix 10.5.0.0/16 `
  --subnet-name $subnet `
  --subnet-prefix 10.5.0.0/24
  --location $location
```

### Step 5: Create a Network Security Group (NSG) and attach it to the subnet

```powershell
# Create a network security group
az network nsg create `
  --resource-group $resourceGroup `
  --name $subnetSecurity
  --location $location

# Attach network security group to virtual network subnet
az network vnet subnet update `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --name $subnet `
  --network-security-group $subnetSecurity
```

### Step 6: Peer spoke vnets to hub and jumpbox

#### Peer vnet-oracle01 - vnet-hub - vnet-jumpbox

```powershell
# Variables
$hubResourceGroup="rg-hub"
$hubVNet="vnet-hub"
$jumpboxResourceGroup="rg-jumpbox"
$jumpboxVNet="vnet-jumpbox"
```

```powershell
az network vnet peering create `
  --name oracle01-to-hub `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$hubResourceGroup/providers/Microsoft.Network/virtualNetworks/$hubVNet" `
  --allow-vnet-access

az network vnet peering create `
  --name hub-to-oracle01 `
  --resource-group $hubResourceGroup `
  --vnet-name $hubVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  
```

```powershell
az network vnet peering create `
  --name oracle01-to-jumpbox `
  --resource-group $resourceGroup `
  --vnet-name $vnet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$jumpboxResourceGroup/providers/Microsoft.Network/virtualNetworks/$jumpboxVNet" `
  --allow-vnet-access

az network vnet peering create `
  --name jumpbox-to-oracle01 `
  --resource-group $jumpboxResourceGroup `
  --vnet-name $jumpboxVNet `
  --remote-vnet "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet" `
  --allow-vnet-access  
```

### Step 7: Create a public IP and a network interface to be used by JumpboxVM01 Virtual Machine

```powershell
# Variables
$publicIP="pip-OracleVM01"
$networkInterface="nic-OracleVM01"
```

```powershell
# Create a public IP
az network public-ip create `
  --resource-group $resourceGroup `
  --name $publicIP `
  --sku Standard `
  --allocation-method Static `
  --location $location
```

```powershell
# Create a network interface
az network nic create `
  --resource-group $resourceGroup `
  --name $networkInterface `
  --vnet-name $vnet `
  --subnet $subnet `
  --network-security-group $subnetSecurity `
  --public-ip-address $publicIP `
  --location $location
```

### Step 8: Create the Oracle VM

#### Generate an SSH key pair to securely connect to the VM, then provision the instance using Azure CLI.

💡 Note:
SSH (Secure Shell) is the most common method for securely accessing Linux VMs.
It encrypts your connection, ensuring that credentials and commands cannot be intercepted.
See [Microsoft Learn: Connect to a Linux VM](https://learn.microsoft.com/en-us/azure/virtual-machines/linux-vm-connect?tabs=Linux) for more details.

```powershell
# Define your SSH key name
$SSH_KEY_NAME="oraclevm01-key-ed"

# Generate SSH key pair
ssh-keygen -t ed25519 -f ~/.ssh/$SSH_KEY_NAME -q -N ""
```

| Parameter                 | Description                                                                                                                                               |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-t ed25519`              | Specifies the key type — **ED25519** is a modern, fast, and secure elliptic-curve algorithm.                                                              |
| `-f ~/.ssh/$SSH_KEY_NAME` | Defines the file path and name for your key pair.<br>📁 **Public key:** `~/.ssh/oraclevm01-key-ed.pub` <br>🔒 **Private key:** `~/.ssh/oraclevm01-key-ed` |
| `-q`                      | Runs quietly without printing progress messages.                                                                                                          |
| `-N ""`                   | Creates a key with an **empty passphrase** (no password prompt when connecting).                                                                          |

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

#### Deploy Oracle VM

```powershell
# Variables
$vmName="OracleVM01"
$osDisk="disk1-OracleVM01"
```
⚠️ **Note:** Replace placeholder values like `<your-admin-username>` with your own before running the commands.

```powershell
# Create the virtual machine
az vm create `
  --resource-group $resourceGroup `
  --name $vmName `
  --location $location `
  --nics $networkInterface `
  --availability-set "" `
  --image "Oracle:Oracle-Linux:ol810-lvm-gen2:8.10.7" `
  --size "Standard_D4s_v3" `
  --admin-username <your-admin-username> `
  --ssh-key-values ~/.ssh/$SSH_KEY_NAME.pub `
  --authentication-type ssh `
  --security-type "TrustedLaunch" `
  --enable-agent true `
  --os-disk-size-gb 64 `
  --os-disk-name $osDisk `  
  --storage-sku Premium_LRS
```  

### Step 9: Allow SSH Access to the Oracle Linux VM

```powershell
# Variables
$rdpRuleName = "AllowSSH"
$sourceIP = "<your source ip>/32"   # Your Public IP or Private subnet range of JumpboxVM01 - 10.1.0.0/24
$priority = 100  # Lower number = higher priority
```

```powershell
# Create NSG rule to allow RDP
az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $subnetSecurity `
  --name $rdpRuleName `
  --priority $priority `
  --protocol Tcp `
  --source-address-prefixes $sourceIP `
  --destination-port-ranges 22 `
  --description "Allow SSH access to Oracle VM from trusted IP"
```

### Step 10: Allow Oracle Database access

```powershell
# Variables
$rdpRuleName = "AllowConnectionToOracle"
$sourceIP = "<your source ip>/32"   # Private subnet range of JumpboxVM01 - 10.1.0.0/24
$priority = 110  # Lower number = higher priority
```

```powershell
# Create NSG rule to allow RDP
az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $subnetSecurity `
  --name $rdpRuleName `
  --priority $priority `
  --protocol Tcp `
  --source-address-prefixes $sourceIP `
  --destination-port-ranges 1521 `
  --description "Allow SSH access to Oracle VM from trusted IP"
```

### Step 11: Connect to Oracle VM

This guide explains how to connect to your Oracle Linux virtual machine from your laptop or from the JumpboxVM01 using SSH (Secure Shell).

The connection requires using the private key you generated earlier when creating your Oracle VM.

💡 Note:
Oracle is not pre-installed on the VM. You’ll use SSH to log into the Linux OS and install Oracle manually.
SSH provides a secure, encrypted command-line connection to your VM.

#### Locate Your SSH Keys in Azure Cloud Shell

When you generated the SSH key in Azure Cloud Shell, two files were created.

```powershell
# List your SSH keys in Cloud Shell:
ls ~/.ssh
```

You should see both key files listed:

| File                    | Description                                       |
| ----------------------- | ------------------------------------------------- |
| `oraclevm01-key-ed`     | Private key - used to authenticate to your VM     |
| `oraclevm01-key-ed.pub` | Public key - uploaded to Azure during VM creation |

#### View the Private Key

Display the private key in Cloud Shell:

```powershell
cat ~/.ssh/oraclevm01-key-ed
```

#### Download the Private Key

- In the Azure Portal, open the Cloud Shell file manager and go to "Manage files".
- Click Download, and download the file oraclevm01-key-ed. 
    - The whole path is /home/batuhan/.ssh/oraclevm01-key-ed
- Move the Private Key to Your SSH Folder

    ```powershell
    # Create the .ssh folder if it doesn't exist
    New-Item -ItemType Directory -Path "$env:USERPROFILE\.ssh" -Force

    # Move your downloaded key file to .ssh folder
    Move-Item "C:\Users\<YourUsername>\Downloads\oraclevm01-key-ed" "$env:USERPROFILE\.ssh\"
    ```

- Secure the Key File: You must set the correct permissions on the key to ensure SSH will accept it.

    ```powershell
    # Run in PowerShell (Admin)
    icacls "$env:USERPROFILE\.ssh\oraclevm01-key-ed" /inheritance:r /grant:r "${env:USERNAME}:R"
    ```

#### Connect to the Oracle VM

- Open Windows Powershell with Admin and run the following command

    💡 Note:
    Replace <oracle-vm-public-ip-or-private-ip> with your actual Oracle VM's public or private IP address

    ```powershell
    #Use the following command to connect from your laptop or Jumpbox:
    ssh -i $env:USERPROFILE\.ssh\oraclevm01-key-ed azadmin@<oracle-vm-public-ip-or-private-ip>
    ```

    ⚠️ Unknown Host Warning: This is a security feature to prevent man-in-the-middle attacks.
    If you're confident you're connecting to your own VM (which you are), simply type: yes then press enter.
    
    If you see the message like above, SSH will add the VM’s fingerprint to your known_hosts file so it won’t ask again next time. 
    If you see the message like below, SSH is warning you it hasn"t seen this host before. You can safely type yes and press Enter. SSH will remember this host in your known_hosts file for future sessions.  

#### Troubleshooting SSH Connection

- If you see connection timed out, It means your VM's port 22 (SSH) is blocked. Go to Azure Portal → VM → Networking → Inbound Port Rules and add:

    | Setting              | Value                              |
    | -------------------- | ---------------------------------- |
    | **Source**           | My IP Address                      |
    | **Destination Port** | 22                                 |
    | **Protocol**         | TCP                                |
    | **Priority**         | 1000 (or lower than any Deny rule) |
    | **Name**             | AllowSSH                           |

### Step 12: Install Oracle on Oracle-Linux 8.10

- Connect from your laptop or JumpboxVM01 using your SSH private key.

    💡 Note:
    Replace <oracle-vm-public-ip-or-private-ip> with your actual Oracle VM's public or private IP address

    ```powershell
    #Use the following command to connect from your laptop or Jumpbox:
    ssh -i $env:USERPROFILE\.ssh\oraclevm01-key-ed azadmin@<oracle-vm-public-ip-or-private-ip>
    ```

- Update packages and install dependencies required for Oracle XE.

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

- Download the RPM package for Oracle Database 21c XE.

    ```powershell
    wget https://download.oracle.com/otn-pub/otn_software/db-express/oracle-database-xe-21c-1.0-1.ol8.x86_64.rpm
    ```

- Install Oracle XE.

    ```powershell
    sudo dnf localinstall -y oracle-database-xe-21c-1.0-1.ol8.x86_64.rpm
    ```

- Run the configuration script to set up the database.

    ```powershell
    sudo /etc/init.d/oracle-xe-21c configure
    ```

    | Prompt            | Example / Default | Description                                  |
    | ----------------- | ----------------- | -------------------------------------------- |
    | **Listener Port** | `1521`            | Oracle network port                          |
    | **APEX Port**     | `5500`            | For Oracle Application Express (web console) |
    | **Passwords**     | `Microsoft123`    | For SYS, SYSTEM, and PDBADMIN                |
    | **Start on Boot** | `Yes`             | Auto-start database on VM boot               |

    After successful configuration, you’ll see the **output**:

    ```text
    Connect to Oracle Database using one of the connect strings:
        Pluggable database: OracleVM01/XEPDB1
        Multitenant container database: OracleVM01
    Use https://localhost:5500/em to access Oracle Enterprise Manager for Oracle Database XE

- Post-Installation Configuration

    Fix permissions, configure the listener, and set Oracle environment variables.

    ```powershell
    # Fix listener log directory permissions
    sudo mkdir -p /opt/oracle/product/21c/dbhomeXE/network/log
    sudo chown -R oracle:oinstall /opt/oracle/product/21c/dbhomeXE/network/log

    # Switch to Oracle user
    sudo su - oracle
    ```

    Create a system-wide profile script under /etc/profile.d to make Oracle environment variables available to all users and sessions, 

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

    Start the listener

    ```powershell
    lsnrctl start
    ```

    Start the Database Instance

    ```powershell
    sqlplus / as sysdba
    ```

    Starting Oracle XE and Opening Pluggable Database

    ```powershell
    # Boots the container database named XE
    # Allocates memory, mounts the database, and opens it for use
    STARTUP;

    # Opens the pluggable database (PDB) named XEPDB1
    # Makes it accessible for connections, queries, and schema operations
    ALTER PLUGGABLE DATABASE XEPDB1 OPEN;

    # Forces the database to re-register with the listener
    # Useful if the listener was started after the database, or if dynamic registration failed
    ALTER SYSTEM REGISTER;
    
    # Exits the SQL*Plus session
    EXIT;
    ```

    Validate Listener Services

    ```powershell
    lsnrctl status
    ```

    ```test
    You should see XE, XEPDB1, and XEXDB under Services Summary.
    ```

    Test SQL*Plus Connection
    ```powershell
    sqlplus system@localhost/XEPDB1
    ```

- Install Git to download sample schemas

    As the `<your-admin-username>` user:

    ```powershell
    sudo dnf install -y git
    git --version
    ```

- Download Oracle Sample Databases

    Clone the official Oracle sample schema repository:

    ```powershell
    cd ~
    git clone https://github.com/oracle-samples/db-sample-schemas.git
    cd db-sample-schemas
    ls
    ```

    💡 Note: You should see the following databases. For more information, review the [document](https://github.com/oracle-samples/db-sample-schemas).
    - CO (Customer Orders) - OLTP 
    - HR (Human Resources) - OLTP
    - SH (Sales History) - OLAP

- Load Sample Databases - customer_orders

    ```powershell
    cd ~/db-sample-schemas/customer_orders
    sqlplus system@localhost/XEPDB1 (or alternate sqlplus system@"localhost:1521/XEPDB1")
    @co_install.sql
    ```
    💡 Note: 

    - Enter password: <set password for CO>
    - Tablespace: Press Enter (defaults to USERS)

    Verify data

    ```powershell
    sqlplus co@localhost/XEPDB1
    SELECT COUNT(*) FROM customers;
    SELECT COUNT(*) FROM orders;
    ```

- Load Sample Databases - sales_history

    ```powershell
    cd ~/db-sample-schemas/sales_history
    sqlplus system@localhost/XEPDB1 (or alternate sqlplus system@"localhost:1521/XEPDB1")
    @sh_install.sql
    ```
    💡 Note: 

    - Enter password: <set password for SH>
    - Tablespace: Press Enter (defaults to USERS)

    Verify data

    ```powershell
    sqlplus sh@localhost/XEPDB1
    SELECT COUNT(*) FROM sales;
    ```

- Load Sample Databases - human_resources

    ```powershell
    cd ~/db-sample-schemas/human_resources
    sqlplus system@localhost/XEPDB1 (or alternate sqlplus system@"localhost:1521/XEPDB1")
    @hr_install.sql
    ```
    💡 Note: 

    - Enter password: <set password for HR>
    - Tablespace: Press Enter (defaults to USERS)

    Verify data

    ```powershell
    sqlplus hr@localhost/XEPDB1
    SELECT COUNT(*) FROM employees;
    ```

- Create a Custom Test User and Table

    Connect as SYSTEM

    ```powershell
    sqlplus system@localhost/XEPDB1 (or alternate sqlplus system@"localhost:1521/XEPDB1")
    ```

    Create a new user and grant permissions:

    ```powershell
    CREATE USER test_user IDENTIFIED BY <password>;
    GRANT CONNECT, RESOURCE TO test_user;
    ALTER USER test_user QUOTA UNLIMITED ON USERS;
    ```

    Connect as new user

    ```powershell
    sqlplus test_user@localhost:1521/XEPDB1
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

- Run queries for the databases

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

You now have a fully configured Oracle XE environment with sample and custom databases.
This setup can be used for migration testing, training, or integration validation with Azure PostgreSQL.