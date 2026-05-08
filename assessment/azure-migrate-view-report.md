# Overview of Azure Migrate reports

Azure Migrate reports provide a summarized view of Migration and modernization opportunities to Azure. They include insights on workload readiness, security, and costs to help you prioritize workloads and make informed migration and modernization decisions. Reports can be generated after successful discovery and inventory enrichment.

Reports provide a summarized view of migration and modernization insights such as:

- **Workload and application summary** across your discovered estate, including web apps, databases, and file servers.
- **Security insights** and **vulnerability summary** for discovered workloads.
- **Utilization summary** for workloads across the estate.
- **Software inventory summary**, including the types of software supporting your applications.
- **Total cost of ownership (TCO)** and **return on investment (ROI)** considerations for migration and modernization scenarios.
- **Value realization of Azure**, including potential savings through Azure benefits, management solutions, and pricing options.
- **Readiness**, **target recommendations**, and **Azure cost insights** for workloads based on the selected migration preference.
- **High-level migration wave plan** to support phased migration planning.
- **Exportable**, **read-only outputs** available in PowerPoint and Excel formats.

## Download the report

- In your **Azure Migrate Project**, go to **Manage -> Reports** and **(3) download** the report.

    ![Create Azure Migrate Project](/images/Azure-Migrate-37.png)

- In azure-migrate folder, you will see the PowerPoint file and Excel files with all the details

    ![Create Azure Migrate Project](/images/Azure-Migrate-Report-01.png)

    ![Create Azure Migrate Project](/images/Azure-Migrate-Report-02.png)

## Review the report

- Open the PowerPoint file.
- Review **Inventory Summary**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-Report-03.png)

    - Total VMs discovered: 47 (These VMs have at least 1 Database instance, or Webapp, or Fileshare or Application)
        - 21 of them are Windows
        - 26 of them are Linux
    - Server distribution by workload type
        - 16 database servers
        - 18 webapp servers
        - 9 fileshare servers (NFS, SMB)
        - 4 other workloads (These are the VMs without Database instance, Webapp, Fileshare and Application. No any workload has been identified.)
    - Servers running both database and webapps
        - 0 servers running both database and webapps
    - Total workloads: 101
        - It is the sum of database instances, webapps, fileshares (number of fileshares not fileshare servers) and servers
        - 16 database instances, 18 webapps, 20 fileshares and 47 servers
    - Server power status
        - Powered ON for 45 servers
        - Powered OFF for 2 servers
    - Dev/Test vs Production workloads
        - 101 workloads are all production
        - This informaiton is coming from workload tags. Review the tag information [here](https://github.com/batuhan-yildiz/azure-databases/blob/main/assessment/azure-migrate-collector.md#explore-the-inventory). If **AzM_Environment** tag is absent on the servers or workloads, they're considered as production workloads by default. If the workloads and servers operate in the dev/test environment, tag them with AzM.Environment: Dev.
    - Servers: 47
        - 21 Windows servers
        - 14 Red Hat Enterprise Linux (RHEL) servers
        - 12 Other Linux (Ubuntu) servers
    - Databases: 16
        - 5 SQL servers
        - 5 MySQL/MariaDB servers
        - 6 PostgreSQL servers
    - Webapps: 18
        - 10 Java Webapp servers
        - 8 ASP.Net servers
    - Out of support workloads: 7


