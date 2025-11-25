# Oracle To Azure Database for PostgreSQL Data migration with Ora2Pg

Oracle to Azure Database for PostgreSQL migration

References:
- [Migrate Oracle to Azure Database for PostgreSQL Using Ora2Pg](https://learn.microsoft.com/en-us/azure/postgresql/migrate/how-to-migrate-oracle-ora2pg)

## Step 1: Install system packages on Ubuntu 22.04

The purpose to install the packages is: 
- build-essential, make, gcc required to compile Perl/DBD modules or Ora2Pg from source.
- libpq-dev needed by DBD::Pg.
- libaio1 is often required by Oracle Instant Client.
- cpanminus (cpanm) makes installing Perl modules easier.

```bash
sudo apt update

sudo apt install -y build-essential make wget cpanminus perl     libdbi-perl libwww-perl libxml-simple-perl libterm-readkey-perl libaio1 gcc unzip libssl-dev libpq-dev libdbd-pg-perl libexpat1-dev postgresql-client

# Check the installed perl version
perl -v
```

### Step 2: Install Oracle Instant Client (required for DBD::Oracle)

1. Open a browser on JumpboxVM01 (Windows) and go to the [Oracle Instant Client Downloads](https://www.oracle.com/database/technologies/instant-client/downloads.html)

2.  Download the Linux x86-64 ZIP files for your version of Oracle:
    - instantclient-basic-linux.x64-23.26.0.0.0.zip → contains runtime libraries to connect to Oracle. ✅ Needed.
    - instantclient-sdk-linux.x64-23.26.0.0.0.zip → contains headers needed to compile DBD::Oracle. ✅ Needed.
    - instantclient-sqlplus-linux.x64-23.26.0.0.0.zip → SQLPlus CLI client.

    💡 Note:
    - Downloaded all these zip files for this exercise.

3. Copy ZIP files from Windows (JumpboxVM01) to Linux (JumpboxVM02)
   - Open PowerShell on JumpboxVM01 without Linux SSH and run:

    ```powershell
    # Adjust paths for where your ZIP files are located
    # 10.1.0.5 is jumpboxvm02 private ip

    # Copy the Basic package
    scp -i $env:USERPROFILE\.ssh\jumpboxvm02-key-ed `
        C:\Users\<your_user>\Downloads\instantclient-basic-*.zip `
        azadmin@10.1.0.5:/home/azadmin/

    # Copy the SDK package
    scp -i $env:USERPROFILE\.ssh\jumpboxvm02-key-ed `
        C:\Users\<your_user>\Downloads\instantclient-sdk-*.zip `
        azadmin@10.1.0.5:/home/azadmin/

    # Copy the SQLPlus package
    scp -i $env:USERPROFILE\.ssh\jumpboxvm02-key-ed `
        C:\Users\<your_user>\Downloads\instantclient-sqlplus-*.zip `
        azadmin@10.1.0.5:/home/azadmin/
    ```

    💡 Note:
    - You are uploading the files to your Linux home directory first (/home/azadmin) because normal users can write there. Then we’ll move them to /opt/oracle/products with sudo.

4. Move files to /opt/oracle/products and unzip on JumpboxVM02 (Linux)

    ```bash
    # Create installation directory
    sudo mkdir -p /opt/oracle/products

    # Move ZIP files from home to products folder
    sudo mv ~/instantclient-basic-linux.*.zip /opt/oracle/products/
    sudo mv ~/instantclient-sdk-linux.*.zip /opt/oracle/products/
    sudo mv ~/instantclient-tools-linux.*.zip /opt/oracle/products/
    sudo mv ~/instantclient-sqlplus-linux.*.zip /opt/oracle/products/

    # Go to products folder
    cd /opt/oracle/products

    # List the files
    ls -l

    # Unzip the packages
    # If it prompts, replace /opt/oracle/products/META-INF/MANIFEST.MF? [y]es, [n]o, [A]ll, [N]one, [r]ename: Select N
    sudo unzip instantclient-basic-linux.*.zip -d /opt/oracle/products
    sudo unzip instantclient-sdk-linux.*.zip -d /opt/oracle/products
    sudo unzip instantclient-sqlplus-linux.*.zip -d /opt/oracle/products
    ```

5. Create a symlink for convenience

   - The symlink lets you refer to /opt/oracle/instantclient in environment variables, making upgrades easier.

    ```bash
    # Check the folder name created by unzip
    ls -l /opt/oracle/products
    # example output: instantclient_23_26

    # Create symlink
    sudo ln -s /opt/oracle/products/instantclient_23_26 /opt/oracle/instantclient
    ```

6. Set environment variables

    ```bash
    # For all users (recommended)
    sudo tee /etc/profile.d/oracle.sh >/dev/null <<'EOF'
    export ORACLE_HOME=/opt/oracle/instantclient
    export LD_LIBRARY_PATH=/opt/oracle/instantclient:$LD_LIBRARY_PATH
    export PATH=/opt/oracle/instantclient:$PATH
    EOF

    # Reload variables
    source /etc/profile.d/oracle.sh

    # Verify environment variables
    echo $ORACLE_HOME
    echo $LD_LIBRARY_PATH
    echo $PATH | grep instantclient
    ```

7. Install Perl DBI modules: DBI, DBD::Oracle, DBD::Pg

- Ora2Pg uses Perl DBI to talk to both Oracle (via DBD::Oracle) and PostgreSQL (DBD::Pg). DBD::Oracle must be compiled against your Instant Client
- If cpanm DBD::Oracle fails, check ORACLE_HOME, LD_LIBRARY_PATH, and that you installed the SDK package of the instant client. You may need sudo apt install libperl-dev if compilation errors appear

    ```powershell
    # install DBD::Oracle (this will compile and link against Instant Client)
    sudo cpanm -n DBD::Oracle
    # If installation fails, erview the log file
    # sudo cat /root/.cpanm/work/1764015156.18281/build.log
    #If the problem is related to environment variables although you set properly, you can try 
    <#
        sudo ORACLE_HOME=/opt/oracle/products/instantclient_23_26 \
        LD_LIBRARY_PATH=/opt/oracle/products/instantclient_23_26:$LD_LIBRARY_PATH \
        PATH=/opt/oracle/products/instantclient_23_26:$PATH \
        cpanm -n DBD::Oracle
    #>

    # Verify DBD::Oracle installed correctly
    perl -MDBD::Oracle -e 'print $DBD::Oracle::VERSION . "\n";'


    # install DBI and Postgres driver
    sudo cpanm DBI DBD::Pg

    # Verify DBD::Pg installed correctly
    perl -MDBD::Pg -e 'print $DBD::Pg::VERSION . "\n";'
    ```

### Step 6: Install Ora2Pg

1. Open a browser on JumpboxVM01 (Windows) and go to the [Ora2Pg](https://github.com/darold/ora2pg/releases) download releases.
2. Download the latest version of Ora2Pg [As of Nov 2025 - Version 25.0](https://github.com/darold/ora2pg/releases/tag/v25.0)
   - ora2pg-25.0.zip
3. Copy ZIP file from Windows (JumpboxVM01) to Linux (JumpboxVM02)
   - Open PowerShell on JumpboxVM01 without Linux SSH and run:

    ```powershell
    # Adjust paths for where your ZIP file is located
    # 10.1.0.5 is jumpboxvm02 private ip

    # Copy the Basic package
    scp -i $env:USERPROFILE\.ssh\jumpboxvm02-key-ed `
        C:\Users\<your_user>\Downloads\ora2pg-25.0.zip `
        azadmin@10.1.0.5:/home/azadmin/
    ```

    💡 Note:
    - You are uploading the files to your Linux home directory first (/home/azadmin) because normal users can write there. Then we’ll move them to /opt/oracle/products with sudo.

4. Move file to /opt/oracle/products and unzip on JumpboxVM02 (Linux)

    ```bash
    # The following directory must exist
    # sudo mkdir -p /opt/oracle/products

    # Move ZIP file from home to products folder
    sudo mv ~/ora2pg-25.0.zip /opt/oracle/products/

    # Go to products folder
    cd /opt/oracle/products

    # List the files
    ls -l

    # Unzip the package
    sudo unzip ora2pg-25.0.zip -d /opt/oracle/products

    # Navigate to the extracted Ora2pg folder
    cd /opt/oracle/products/ora2pg-25.0 # adjust the path if needed

    # List the files
    ls -l    
    ```

5. Installation Proces

   - Install ora2pg

    ```powershell
    # Create a working folder in your home directory
    cd ~
    mkdir -p ora2pg
    # copies all files and subdirectories inside ora2pg and list the files
    cp -r /opt/oracle/products/ora2pg-25.0/* ~/ora2pg/
    cd ~/ora2pg
    ls -l 

    # Prepare the build environment. Reads the Ora2Pg Perl Makefile template and generates a Makefile based on your system’s Perl modules and libraries.
    perl Makefile.PL

    # Compiles/builds the software according to the Makefile.
    # For Ora2Pg, this mostly sets up Perl modules and scripts so they are ready to install.
    make

    # Install system-wide (requires sudo)
    # This copies the Ora2Pg scripts and binaries into system-wide directories like /usr/local/bin so that you can run ora2pg from anywhere on the system
    # sudo is needed because writing to system directories requires root permissions.
    # After this, you can just type ora2pg in any terminal and it will work.
    sudo make install
    ```
   - Verify your installation

    ```powershell
    ora2pg --version
    ```

### Step 7: Quick environment validation

```powershell
# check the tools
perl -v
cpanm --version
psql --version
ora2pg --version

# load the modules. If the module loads successfully, it prints the confirmation message.
perl -MDBI -e "print 'DBI OK'"
perl -MDBD::Oracle -e "print 'Oracle OK'"
perl -MDBD::Pg -e "print 'PostgreSQL OK'"
```

### Step 8: Test connection

```powershell
# Update the conf file
cd ~/ora2pg
cp ora2pg.conf.dist ora2pg.conf
```

#### Configure Oracle Connection

```powershell
# Edit ora2pg.conf file and update the following parameters
# It has been copied in previous step

nano ~/ora2pg/ora2pg.conf

ORACLE_HOME     /opt/oracle/instantclient
ORACLE_DSN      dbi:Oracle:host=10.4.0.4;service_name=XEPDB1;port=1521
ORACLE_USER     migration_user
ORACLE_PWD      migration_password
#SCHEMA		    SCHEMA_NAME
SCHEMA          your_schema
```

Search for a string (like ORACLE_DSN)

- Press: Ctrl + W (Where Is)
- Nano will prompt: Search:
- Type PG_DSN and press Enter
- Nano will jump to the first match
- Press Ctrl + W again and Enter to find the next match

Save and exit:
- Press Ctrl + O → write out/save
- Press Enter → confirm the filename
- Press Ctrl + X → exit nano

```powershell
# Test Connection
ora2pg -t SHOW_VERSION -c ~/ora2pg/ora2pg.conf
```

- -t SHOW_VERSION → Test the Oracle connection by retrieving database version
- -c ora2pg.conf → Uses your configuration file
- If the connection works, you’ll see the Oracle version printed.

#### Configure PostgreSQL Connection

```powershell
# Edit ora2pg.conf file and update the following parameters
# It has been copied in previous step

 nano ~/ora2pg/ora2pg.conf

PG_DSN dbi:Pg:dbname=customer_orders_ora2pg;host=azpgsqldevtest01.postgres.database.azure.com;port=5432
PG_USER migration_user
PG_PWD migration_password
#PG_SCHEMA
PG_SCHEMA public
```

```powershell
# Test Connection
# There is no direct connection test
# You can use
<#
psql "host=azpgsqldevtest01.postgres.database.azure.com dbname=customer_orders_ora2pg user=migration_user password=migration_password sslmode=require"

SELECT version();
#>
```

### Step 9: Initialize Project (Create an Ora2Pg Project Folder)

```powershell
cd ~
mkdir -p Migration
```

```powershell
ora2pg --project_base ~/Migration --init_project modernization_project_01
```

```text
Creating project modernization_project_01.
/home/azadmin/Migration/modernization_project_01/
        schema/
                dblinks/
                directories/
                functions/
                grants/
                mviews/
                packages/
                partitions/
                procedures/
                sequences/
                sequence_values/
                synonyms/
                tables/
                tablespaces/
                triggers/
                types/
                views/
        sources/
                functions/
                mviews/
                packages/
                partitions/
                procedures/
                triggers/
                types/
                views/
        data/
        config/
        reports/

Generating generic configuration file
Creating script export_schema.sh to automate all exports.
Creating script import_all.sh to automate all imports.
```

💡 Note:
The sources/ directory contains the Oracle code. The schema/ directory contains the code ported to PostgreSQL. And the reports/ directory contains the HTML reports and the migration cost assessment.

After the project structure is created, a generic config file is created. Define the Oracle database connection and the relevant config parameters in the config file.

### Step 10: Assess Oracle Schema

```powershell
# Backup the ora2pg.conf file
cp ora2pg.conf ora2pg_backup.conf

# Edit ora2pg.conf file and update the following parameters
nano ~/Migration/modernization_project_01/config/ora2pg.conf

# Oracle parameters
ORACLE_HOME     /opt/oracle/instantclient
ORACLE_DSN      dbi:Oracle:host=10.4.0.4;service_name=XEPDB1;port=1521
ORACLE_USER     migration_user
ORACLE_PWD      migration_password
#SCHEMA		    SCHEMA_NAME
SCHEMA          your_schema

# PGSQL parameters
PG_DSN dbi:Pg:dbname=customer_orders_ora2pg;host=azpgsqldevtest01.postgres.database.azure.com;port=5432
PG_USER migration_user
PG_PWD migration_password
#PG_SCHEMA
PG_SCHEMA public
```

Search for a string (like ORACLE_DSN)

- Press: Ctrl + W (Where Is)
- Nano will prompt: Search:
- Type PG_DSN and press Enter
- Nano will jump to the first match
- Press Ctrl + W again and Enter to find the next match

Save and exit:
- Press Ctrl + O → write out/save
- Press Enter → confirm the filename
- Press Ctrl + X → exit nano

```powershell
# full assessment - text
ora2pg -t SHOW_REPORT -c ~/Migration/modernization_project_01/config/ora2pg.conf --estimate_cost --cost_unit_value 10 > ~/Migration/modernization_project_01/reports/assessment.txt

# full assessment - html
ora2pg -t SHOW_REPORT -c ~/Migration/modernization_project_01/config/ora2pg.conf --estimate_cost --cost_unit_value 10 --dump_as_html > ~/Migration/modernization_project_01/reports/assessment.html
```

#### Review the assessment result

```powershell
scp -i $env:USERPROFILE\.ssh\jumpboxvm02-key-ed `
    azadmin@10.1.0.5:/home/azadmin/Migration/modernization_project_01/reports/* `
    C:\Users\<your_user>\Downloads\
```

### Step 11: Export Oracle objects (schema)

There are 2 options:

#### Option 1: (Export all objects)

```powershell
cd ~/Migration/modernization_project_01
./export_schema.sh
```

#### Option 2: (Export individual objects)

```bash
# Set Migration Workspace (Home Folder)
namespace="/home/azadmin/Migration/modernization_project_01"
# if needed, set the migration home folder as an environment variable
# export namespace="/home/azadmin/Migration/modernization_project_01"

ora2pg -p -t DBLINK     -o dblink.sql      -b $namespace/schema/dblinks      -c $namespace/config/ora2pg.conf
ora2pg -p -t DIRECTORY  -o directory.sql   -b $namespace/schema/directories  -c $namespace/config/ora2pg.conf
ora2pg -p -t FUNCTION   -o functions2.sql  -b $namespace/schema/functions    -c $namespace/config/ora2pg.conf
ora2pg -p -t GRANT      -o grants.sql      -b $namespace/schema/grants       -c $namespace/config/ora2pg.conf
ora2pg -p -t MVIEW      -o mview.sql       -b $namespace/schema/mviews       -c $namespace/config/ora2pg.conf
ora2pg -p -t PACKAGE    -o packages.sql    -b $namespace/schema/packages     -c $namespace/config/ora2pg.conf
ora2pg -p -t PARTITION  -o partitions.sql  -b $namespace/schema/partitions   -c $namespace/config/ora2pg.conf
ora2pg -p -t PROCEDURE  -o procs.sql       -b $namespace/schema/procedures   -c $namespace/config/ora2pg.conf
ora2pg -p -t SEQUENCE   -o sequences.sql   -b $namespace/schema/sequences    -c $namespace/config/ora2pg.conf
ora2pg -p -t SYNONYM    -o synonym.sql     -b $namespace/schema/synonyms     -c $namespace/config/ora2pg.conf
ora2pg -p -t TABLE      -o table.sql       -b $namespace/schema/tables       -c $namespace/config/ora2pg.conf
ora2pg -p -t TABLESPACE -o tablespaces.sql -b $namespace/schema/tablespaces  -c $namespace/config/ora2pg.conf
ora2pg -p -t TRIGGER    -o triggers.sql    -b $namespace/schema/triggers     -c $namespace/config/ora2pg.conf
ora2pg -p -t TYPE       -o types.sql       -b $namespace/schema/types        -c $namespace/config/ora2pg.conf
ora2pg -p -t VIEW       -o views.sql       -b $namespace/schema/views        -c $namespace/config/ora2pg.conf
```

Parameters:
- -p: Preview / Print. Shows SQL commands that would be executed. In the context of exporting objects, it’s often used for testing without applying changes directly to the database.
- -t: ype of object to export. Examples: TABLE, VIEW, FUNCTION, PACKAGE, DBLINK, SEQUENCE, etc. Determines what Ora2Pg will extract from Oracle.
- -o: Output filename for the SQL export. For example, -o dblink.sql will write the exported DBLINK definitions to a file named
- -b: Base directory where Ora2Pg writes the exported file(s). This is a folder path. If the folder doesn’t exist, Ora2Pg will try to create it. Example: -b $namespace/schema/dblinks will put dblink.sql inside that folder.
- -c: Configuration file to use. The path to your ora2pg.conf that contains Oracle/PG connection details and other migration settings.

For example: 
- Exports DBLINK objects from Oracle.
- Writes them to dblink.sql.
- Saves the file under $namespace/schema/dblinks.
- Uses your configuration file $namespace/config/ora2pg.conf.
- With -p, it prints or previews the export without affecting the database.

### Step 12: Import Oracle objects (schema) into Azure Database for PostgreSQL

There are 2 options:

#### Option 1: Import all objects (schema)

Use import_all.sh to import all schame files interactively.

```powershell
export PGPASSWORD=YourPasswordHere
cd ~/Migration/modernization_project_01
./import_all.sh -d customer_orders_ora2pg -U azpgsqladmin -h azpgsqldevtest01.postgres.database.azure.com -o azpgsqladmin -s -y
```

```text
    echo "    -a             import data only"
    echo "    -b filename    SQL script to execute just after table creation to fix database schema"
    echo "    -d dbname      database name for import"
    echo "    -D             enable debug mode, will only show what will be done"
    echo "    -e encoding    database encoding to use at creation (default: UTF8)"
    echo "    -f             force no check of user and database existing and do not try to create them"
    echo "    -h hostname    hostname of the PostgreSQL server (default: unix socket)"
    echo "    -i             only load indexes, constraints and triggers"
    echo "    -I             do not try to load indexes, constraints and triggers"
    echo "    -j cores       number of connection to use to import data or indexes into PostgreSQL"
    echo "    -n schema      comma separated list of schema to create"
    echo "    -o username    owner of the database to create"
    echo "    -p port        listening port of the PostgreSQL server (default: 5432)"
    echo "    -P cores       number of tables to process at same time for data import"
    echo "    -s             import schema only, do not try to import data"
    echo "    -t export      comma separated list of export type to import (same as ora2pg)"
    echo "    -U username    username to connect to PostgreSQL (default: peer username)"
    echo "    -x             import indexes and constraints after data"
    echo "    -y             reply Yes to all questions for automatic import"
```

#### Option 2: Import individual objects (schema)

Load generated DDL files individually

```powershell
psql -f $namespace/schema/sequences/sequence.sql -h server1-server.postgres.database.azure.com -p 5432 -U username@server1-server -d database -L $namespace/schema/sequences/create_sequence.log

psql -f $namespace/schema/tables/table.sql -h server1-server.postgres.database.azure.com -p 5432 -U username@server1-server -d database -L $namespace/schema/tables/create_table.log
```

### Step 13: Extract the data from Oracle and import into PostgreSQL

```powershell
# Set Migration Workspace (Home Folder)
namespace="/home/azadmin/Migration/modernization_project_01"
```

There are 2 options

#### Extract data into file(s) then import into Azure Database for PostgreSQL

**Extract data into file(s)**

```powershell
# Edit configuration file
nano $namespace/config/ora2pg.conf

# Make the parameters have been set as below
<#
OUTPUT output.sql
OUTPUT_DIR /home/azadmin/Migration/modernization_project_01/data
FILE_PER_TABLE 1
DATA_LIMIT 0
#PG_DSN         <pg_dsn>
#PG_USER                <pgusername>
#PG_PWD         <pg_password>
#>

```powershell
# Extract the actual data from Oracle tables and prepare it for migration to PostgreSQL.
ora2pg -t COPY -o data.sql -b $namespace/data -c $namespace/config/ora2pg.conf
```

- If you dont set the export directory **-b $namespace**, it will use OUTPUT_DIR from config file, as export directory.
- If you don't pass -o data.sql, it will use OUTPUT from config file to create the file. In this example, it will be output.sql.

Parameters in the command:

- -t COPY: Extract data from Oracle and generate PostgreSQL COPY statements
- -o data.sql: Write the output into a file named data.sql. This overrides OUTPUT in the configuration file.
- -b $namespace/data: Export directory. All output files will be generated inside the directory.
- -c $namespace/config/ora2pg.conf: Config file

Parameter in the config file:
- FILE_PER_TABLE 1: Ora2Pg will produce one output file per table instead of one combined file. This is useful when tables are very large or when you want to parallelize loading. If you want a single file, set FILE_PER_TABLE to 0 or remove the option.
- DATA_LIMIT 0: Controls chunking of large data files
  - DATA_LIMIT = 0 → unlimited (no splitting)
  - DATA_LIMIT = 2000000 → split output every 2M rows. It will generate multiple files per table (table_001.sql, table_002.sql)

Output of the command: 
- It will generate one file per table because you set FILE_PER_TABLE 1.
- It will also generate one big data.sql because you set -o data.sql -b $namespace/data.
    - CUSTOMERS_data.sql
    - INVENTORY_data.sql
    - ORDERS_data.sql
    - ORDER_ITEMS_data.sql
    - PRODUCTS_data.sql
    - SHIPMENTS_data.sql
    - STORES_data.sql
    - data.sql

**Import data into PostgreSQL**

```powershell
# Import data into PostgreSQL

psql -h azpgsqldevtest01.postgres.database.azure.com \
    -p 5432 \
    -U azpgsqladmin \
    -d customer_orders_ora2pg \
    -f $namespace/data/CUSTOMERS_data.sql
```
💡 Note:
- -l $namespace/data/CUSTOMERS_data.log parameter is not a valid psql option anymore. In fact,  has never been for logging, historically it was used to list databases (), not to specify a log file.
- If you want to log the output, then you can use 
    - `> $namespace/data/CUSTOMERS_data.log 2>&1`

#### Extract data directly into into Azure Database for PostgreSQL

```powershell
# Edit configuration file to enable PG_DSN, PG_USER and PG_PWD
nano $namespace/config/ora2pg.conf

# Make the parameters have been set as below
<#
OUTPUT output.sql
OUTPUT_DIR /home/azadmin/Migration/modernization_project_01/data
FILE_PER_TABLE 1
DATA_LIMIT 0
PG_DSN         <pg_dsn>
PG_USER                <pgusername>
PG_PWD         <pg_password>
#>

```powershell
# Extract the actual data from Oracle tables and import it into PostgreSQL.
ora2pg -t COPY -c $namespace/config/ora2pg.conf
```

💡 Note:
- Even if you pass `-o data.sql` and/or `-b $namespace/data`, the COPY process will ignore them. Once it sees PGSQL paramaters are enabled and set in configration file, it will directly import into PGSQL


