# Discover servers and workloads using Azure Migrate collector

Azure Migrate collector is a discovery tool to quickly discover servers and workloads across your IT estate without direct Azure connectivity. You deploy the collector on a Windows Server to scan VMware environments and physical or virtual servers. 

Azure Migrate collector can discover your VMware estate or individual Windows and Linux servers running on any hypervisor or public cloud. You can collect server configurations, performance metrics, installed software, SQL Server and PostgreSQL database instances, and web apps (.NET on IIS and Java on Tomcat). With no Azure connectivity required, you can scan the estate locally and upload data securely, saving time and avoiding complex networking or access approval requirements.

- Connectivity: Offline discovery
- Time to discovery: Quick to setup
- Assessment types: Lift and Shift, Modernize
- Performance based right-sized targets: Yes
- Guest and workload discovery: Yes
- Identify server dependencies: No
- Execute migrations: No

## Azure Migrate Collector deployment overview

- Create a new [Azure Migrate project](https://learn.microsoft.com/en-us/azure/migrate/quickstart-create-project?view=migrate)
- Download the Azure Migrate Collector
- Run the installer script and configure the collector
    - Provide vCenter credentials
    - Provide guest and database credentials
    - Validate credentials
    - Start data collection
    - Review collected data and diagnose errors by fixing network access issues, modifying user privileges, or adding new credentials
    - After addressing issues, enable Incremental data to attempt collection only on workloads where previous attempts failed
    - Export collected data
- Upload the collected data to an Azure Migrate project
- Enrich the discovery inventory
    - Apply custom tags like department, business unit, scope etc.
    - Apply reserved tags like dev, test, retain, retire
- After upload is successful, create business cases and assessments
- Import more inventory if needed

## Review before you start

- Set up the collector on a jump-box virtual machine (VM)
- Target environment:
    - vCenter
    - All versions of Windows and Linux
    - Webapps, SQL and PostgreSQL
- Prepare vCenter, guest & database accounts
    - **vCenter account:** Read only and guest operations (To collect server configurations & performance data of VMware machines.)
    - **Windows:** Domain account or administrator* account (To collect installed software, SQL & PostgreSQL database instance and web apps data.)
    - **Linux:** Root account* (To collect installed software, SQL & PostgreSQL database instance and web apps data.)
    - **SQL Server:** Domain account with SQL permissions (To collect SQL readiness data)
    - **Note for \*:** You can set up custom least privileged Windows, Linux, and SQL accounts by referring [this article](https://learn.microsoft.com/en-us/azure/migrate/best-practices-least-privileged-account?view=migrate).
- Azure Migrate Owner role is required to create Azure Migrate Project
- Follow all the [prerequisites](https://learn.microsoft.com/en-us/azure/migrate/how-to-discover-using-collector?view=migrate)

## Create a new Azure Migrate project

- In the Azure portal, search for Azure Migrate.
- In Services, select **Azure Migrate**.
- In **Get started**, select **Create project**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-01.png)

- In **Create project**, select the Azure subscription and resource group. Create a resource group if you don't have one.
- In **Project Details**, specify the project name, the geography in which you want to create the project and connectivity method. 
    - Review supported geographies for [public](https://learn.microsoft.com/en-us/azure/migrate/supported-geographies?view=migrate#public-cloud) and [government clouds](https://learn.microsoft.com/en-us/azure/migrate/supported-geographies?view=migrate#azure-government).
    - Create a project with [private endpoint connectivity](https://learn.microsoft.com/en-us/azure/migrate/discover-and-assess-using-private-endpoints?view=migrate#create-a-project-with-private-endpoint-connectivity).
    - Create a project in a [specific region](https://learn.microsoft.com/en-us/azure/migrate/quickstart-create-project?view=migrate#create-a-project-in-a-specific-region).
- Select **Create**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-02.png)

- In **All projects**, review your **Azure Migrate Project**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-03.png)

## Download the Azure Migrate Collector

- In the Azure Migrate, select **All Projects -> your Azure Migrate Project**.
- In your Azure Migrate Project, select **Start discovery -> Using collector**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-04.png)

- In Discover, select **Download**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-05.png)

- Alternatively, download the Azure Migrate collector installer [here](https://aka.ms/Migrate/DownloadCollector). 

## Run the installer script on your jump-box

- Connect to your jump-box.
- Copy and extract AzureMigratecollector.zip file to your jump-box.
- Launch PowerShell with administrative privileges.
- Change the directory to the extracted folder.
- Run the installer script:

    ```powershell
    .\AzureMigratecollector.ps1
    ```
- For the first installation, select the fresh (f) option.

    ![Create Azure Migrate Project](/images/Azure-Migrate-06.png)


