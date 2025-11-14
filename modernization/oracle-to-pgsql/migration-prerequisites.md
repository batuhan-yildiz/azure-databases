# Oracle To Azure Database for PostgreSQL Pre-Migration

- RDP to JumpboxVM01 and open VS Code
- install the required extensions in VS Code
  - Oracle SQL Developer Extension for VS Code
- Create a new connection to Oracle database server and create a read-only migration account
-  Create a new connection to Azure Database for PostgreSQL

## RDP to JumpboxVM01 and test Oracle database server connection 

- RDP to JumpboxVM01
- Open VS Code
- Install Oracle SQL Developer Extension for VS Code if not installed
- Create a new connection to Oracle
  - Connection Name: OracleVM01-SYSTEM
  - Username: system
  - Password: <system_password>
  - Hostname: <OracleVM01_private_ip>
  - Port: <OracleVM01_database_server_port_number>
  - Service Name: XEPDB1
  - Test and Save connection. If connection fails, Review port in OracleVM01 Server Firewall, OracleVM01 Network Security Group.

## Create a read-only migration account in Oracle database server

Connect to Oracle Server on JumpboxVM01 with VS Code

Open a new SQL Worksheet and run the following commands

```sql
--CLEANUP (DROP EXISTING MIGRATION USER IF EXISTS)
BEGIN
   EXECUTE IMMEDIATE 'DROP USER MIGRATIONUSER CASCADE';
EXCEPTION
   WHEN OTHERS THEN
      IF SQLCODE = -01918 THEN 
         DBMS_OUTPUT.PUT_LINE('MIGRATIONUSER does not exist — skipping drop.');
      ELSE 
         RAISE;
      END IF;
END;

-- CREATE OR UPDATE MIGRATION ROLE
BEGIN
   EXECUTE IMMEDIATE 'CREATE ROLE MIGRATION_ROLE';
EXCEPTION
   WHEN OTHERS THEN
      IF SQLCODE = -01921 THEN 
         DBMS_OUTPUT.PUT_LINE('MIGRATION_ROLE already exists — skipping creation.');
      ELSE 
         RAISE;
      END IF;
END;

-- Grant login and reading metadata to MIGRATION_ROLE
GRANT CREATE SESSION TO MIGRATION_ROLE;
GRANT SELECT_CATALOG_ROLE TO MIGRATION_ROLE;

-- CREATE MIGRATION USER AND ASSIGN ROLE
CREATE USER <migration_user> IDENTIFIED BY "<migration_user_password>";
GRANT CREATE SESSION TO MIGRATIONUSER;
GRANT MIGRATION_ROLE TO MIGRATIONUSER;
ALTER USER MIGRATIONUSER DEFAULT ROLE ALL;

-- CREATE SYNONYMS FOR DBA_* VIEWS (ALL SYSTEM)
-- Why: Tools like VSIX / Ora2PG query DBA_OBJECTS, DBA_TABLES, DBA_VIEWS
--      but MIGRATIONUSER does not have DBA privileges. 
--      We create synonyms owned by MIGRATIONUSER while connected as SYSTEM.
BEGIN
   EXECUTE IMMEDIATE 'CREATE SYNONYM MIGRATIONUSER.DBA_OBJECTS FOR ALL_OBJECTS';
EXCEPTION WHEN OTHERS THEN IF SQLCODE = -00955 THEN NULL; ELSE RAISE; END IF;
END;

BEGIN
   EXECUTE IMMEDIATE 'CREATE SYNONYM MIGRATIONUSER.DBA_TABLES FOR ALL_TABLES';
EXCEPTION WHEN OTHERS THEN IF SQLCODE = -00955 THEN NULL; ELSE RAISE; END IF;
END;

BEGIN
   EXECUTE IMMEDIATE 'CREATE SYNONYM MIGRATIONUSER.DBA_VIEWS FOR ALL_VIEWS';
EXCEPTION WHEN OTHERS THEN IF SQLCODE = -00955 THEN NULL; ELSE RAISE; END IF;
END;

-- GRANT OBJECT ACCESS FOR TARGET SCHEMA
DECLARE
   v_owner VARCHAR2(30) := 'CO';  -- Replace with actual schema name
BEGIN
   -- Tables
   FOR t IN (SELECT table_name FROM all_tables WHERE owner = v_owner) LOOP
      EXECUTE IMMEDIATE 'GRANT SELECT ON ' || v_owner || '.' || t.table_name || ' TO MIGRATION_ROLE';
   END LOOP;

   -- Views
   FOR v IN (SELECT view_name FROM all_views WHERE owner = v_owner) LOOP
      EXECUTE IMMEDIATE 'GRANT SELECT ON ' || v_owner || '.' || v.view_name || ' TO MIGRATION_ROLE';
   END LOOP;

   -- Sequences
   FOR s IN (SELECT sequence_name FROM all_sequences WHERE sequence_owner = v_owner) LOOP
      EXECUTE IMMEDIATE 'GRANT SELECT ON ' || v_owner || '.' || s.sequence_name || ' TO MIGRATION_ROLE';
   END LOOP;

   -- Procedures, Functions, Packages
   FOR p IN (SELECT object_name FROM all_objects WHERE owner = v_owner AND object_type IN ('PROCEDURE','FUNCTION','PACKAGE')) LOOP
      EXECUTE IMMEDIATE 'GRANT EXECUTE ON ' || v_owner || '.' || p.object_name || ' TO MIGRATION_ROLE';
   END LOOP;

   -- Optional: Object types (UDTs)
   FOR t IN (SELECT type_name FROM all_types WHERE owner = v_owner) LOOP
      EXECUTE IMMEDIATE 'GRANT EXECUTE ON ' || v_owner || '.' || t.type_name || ' TO MIGRATION_ROLE';
   END LOOP;

   -- Optional: Materialized views
   FOR m IN (SELECT mview_name FROM all_mviews WHERE owner = v_owner) LOOP
      EXECUTE IMMEDIATE 'GRANT SELECT ON ' || v_owner || '.' || m.mview_name || ' TO MIGRATION_ROLE';
   END LOOP;
END;

-- VERIFICATION
-- Connect as MIGRATIONUSER and run:
SELECT USER FROM dual;
SELECT COUNT(*) FROM ALL_TABLES WHERE OWNER = 'CO';
SELECT OBJECT_NAME, OBJECT_TYPE FROM ALL_OBJECTS WHERE OWNER = 'CO';
SELECT COUNT(*) FROM DBA_OBJECTS;
SELECT * FROM CO.CUSTOMERS WHERE ROWNUM <= 5;
```

## RDP to JumpboxVM01 and test Azure Database for PostgreSQL database server connection 

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
  - Click on Advanced
    - Port: 5432
  - Test and Save connection. 