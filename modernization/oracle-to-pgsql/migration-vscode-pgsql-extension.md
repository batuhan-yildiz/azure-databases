# Oracle To Azure Database for PostgreSQL Migration with VS Code

## Step 1: Add PostgreSQL connection

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
    - Connection Timeout: 15 (default)- 
  - Test and Save connection. 

## Step 2: Create a new migration project

- New PostgreSQL migration project
  - Project Name: - Oracle-to-PGSQL Migration01
  - Click on **Next: Oracle Connection** button

- Connect to Oracle
  - Oracle Hostname: 10.5.0.4
  - Oracle Port: 1521
  - Oracle SID or Service Name: XEPDB1
  - Oracle Username <migration_user>
  - Oracle Password: <migration_password>
  - Schemas: Click on **Load Schemas** button and select the schema you want to modernize
  - Click on **Next: PostgreSQL Connection** button

- Choose a scratch PostgreSQL database
  - PostgreSQL Connection: pgsql-azpgsqlprod01 (It was created in previous step)
  - PostgreSQL Database: Click on **Load Databases** button and select the database you want to write into
  - Click on **Next: Language Model Configuration** button

- Choose a Language Model
  - Language Model Type: Custom OpenAI
  - OpenAI Endpoint: <copy_from_custom_openai_service> 
    <https://<your_custom_openai_endpoint_name>.privatelink.openai.azure.com/   -- Review custom openai creation link for details> 
  - OpenAI API Key: <copy_from_custom_openai_service>
  - Language Model: GPT-4.1
  - Click on **Test Open AI Connection** to test the model
  - Once you see "[All]: OpenAI connection successful" in the output window, click on **Create Migration Project**.

- Oracle-to-PGSQL Migration01 project main view
  - Schema Migration: Step 1 - Convert database code and scripts to PostgreSQL - Click on **Migrate** button.
    - Review the Schema Migration
  - Schema Review
  - Data Migration
  - Application Migration