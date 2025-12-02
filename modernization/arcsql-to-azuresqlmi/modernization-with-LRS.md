# Modernize Arc-enabled SQL Server to Azure SQL Managed Instance with LRS

Learn how to migrate your SQL Server instance enabled by Azure Arc to Azure SQL Managed Instance through Managed Instance Link.

After your SQL Server instance is enabled by Azure Arc, you can assess your SQL Server data estate to identify an optimal SQL Managed Instance configuration. Then you can migrate your SQL Server databases to SQL Managed Instance directly from the Azure portal.

We will focus on Log Replay Service (LRS) which is Cloud-managed log-shipping style migration that uses backups uploaded to Blob Storage and replaying logs on the MI. Simpler network requirements, works well for many migrations.