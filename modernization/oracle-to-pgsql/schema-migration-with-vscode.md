# Oracle To Azure Database for PostgreSQL Schema Migration with VS Code

## Step 1: Install PostgreSQL Extension and add PostgreSQL connection

- RDP to JumpboxVM01
- Open VS Code
- Install PostgreSQL Extension for VS Code if not installed
- Create a new connection to Azure Database for PostgreSQL
  - Server Name: <postgresql_endpoint_from_azure_portal>
  - Authentication Type: Password
  - User Name: <administrator_login_from_azure_portal_PostgreSQL>
  - Password: <your-admin-password>
  - Database Name: customer_orders (or select whatever user database you have for test)
  - Connection Name: PGSQLMigration
  - Server Group: Servers
  - Click on **Advanced** button
    - Port: 5432
    - Application Name: vscode-pgsql (default)
    - Connection Timeout: 15 (default)
  - Test and Save connection. 

## Step 2: Create a new migration project

- Click on PostgreSQL in VS Code
- Click on **Open Migration Project** Under Migrations
  - New Oracle to PostgreSQL migration project
    - A migration project uses AI to help you convert database schema (DDL) and application files to PostgreSQL.
    - To create a new migration project, you'll first need to open a project or workspace.
  - Click on **Open Workspace Project** button
  - Select C:\Projects as migration project folder. You can select any folder.
  - If the migration project did not start, right click on the empty space under Projects and click on **Open Migration Project**

- New Oracle to PostgreSQL migration project
  - Project Name: Oracle-to-PGSQL Migration01
  - Click on **Next: Oracle Connection** button

- Connect to Oracle
  - Oracle Hostname: 10.4.0.4
  - Oracle Port: 1521
  - Oracle SID or Service Name: XEPDB1
  - Oracle Username <migration_user>
  - Oracle Password: <migration_password>
  - Schemas: Click on **Load Schemas** button and select the schema you want to modernize
  - Click on **Next: PostgreSQL Connection** button

- Choose a scratch PostgreSQL database
  - Click on **Refresh Profiles**
  - PostgreSQL Connection: azpgsqldevtest01 (It was created in previous step)
  - PostgreSQL Database: Click on **Load Databases** button and select the database you want to write into
  - Click on **Next: Language Model Configuration** button

- Choose a Language Model
  - Language Model Type: Custom OpenAI
  - OpenAI Endpoint: <copy_from_custom_openai_service>
  - OpenAI API Key: <copy_from_custom_openai_service_key_1>
  - Language Model: GPT-4.1
  - Click on **Test OpenAI Connection** to test the model
  - Once you see "OpenAI connection successful" in the output window, click on **Create Migration Project**.

- Oracle-to-PGSQL Migration01 project main view

    > ⚠️ Note:
    Choose your Open AI model configuration. 500000 TPM (Tokens Per Minute) recommended for optimal performance.

  - Schema Migration: 
    - Step 1 - Convert database schema to PostgreSQL. Click on **Migrate** button.
    - Review the Schema Migration progress
  - Schema Review
    - Step 2: Review and finalize migration tasks.
    - Review Oracle to PostgreSQL Migration Report
  - Application Migration
    - Step 3: Select and convert Oracle scripts to PostgreSQL.