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

- In your **Azure Migrate Project**, go to **Manage -> Reports**, find your report and then select **Download** to download the report.

    ![Create Azure Migrate Project](/images/Azure-Migrate-37.png)

- In the **azure-migrate** folder,  you will find the generated PowerPoint and Excel files containing detailed insights.

    ![Create Azure Migrate Project](/images/Azure-Migrate-Report-01.png)

    ![Create Azure Migrate Project](/images/Azure-Migrate-Report-02.png)

## Review the report

- Open the PowerPoint file.
- Review the **Inventory Summary**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-Report-03.png)

    - **Total VMs discovered:** 47 (These VMs have at least one database instance, web app, file share or application)
        - 21 are Windows
        - 26 are Linux
    - **Server distribution by workload type**
        - 16 database servers
        - 18 web app servers
        - 9 file share servers (NFS, SMB)
        - 4 other workloads (VMs without a database instance, web app, file share, or application — no workload has been identified)
    - **Servers running both database and web apps**
        - 0 servers running both database and web apps
    - **Total workloads:** 101
        - This is the sum of database instances, web apps, file shares (number of file shares, not file share servers), and servers
        - 16 database instances, 18 web apps, 20 file shares and 47 servers
    - **Server power status**
        - 45 servers are powered on
        - 2 servers are powered off
    - **Dev/Test vs Production workloads**
        - All 101 workloads are classified as production
        - This information is derived from workload tags. Review the tagging details [here](azure-migrate-collector.md#explore-the-inventory). 
            - If **AzM_Environment** tag is absent on the servers or workloads, they're considered as production workloads by default
            - If the workloads and servers operate in the dev/test environment, tag them with AzM.Environment: Dev.
    - **Servers:** 47
        - 21 Windows servers
        - 14 Red Hat Enterprise Linux (RHEL) servers
        - 12 Other Linux (Ubuntu) servers
    - **Databases:** 16
        - 5 SQL servers
        - 5 MySQL/MariaDB servers
        - 6 PostgreSQL servers
    - **Web apps:** 18
        - 10 Java web app servers
        - 8 ASP.Net servers
    - **Out of support workloads:** 7

## Additional resources

- [Discover servers and workloads using Azure Migrate collector](azure-migrate-collector.md)


