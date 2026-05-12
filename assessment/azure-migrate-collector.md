# Discover servers and workloads using Azure Migrate Collector

The Azure Migrate Collector is a discovery tool used to quickly discover servers and workloads across your IT estate without requiring direct Azure connectivity. You deploy the collector on a Windows Server to scan VMware environments as well as physical or virtual servers.

The Azure Migrate Collector can discover your VMware estate or individual Windows and Linux servers running on any hypervisor or public cloud. It collects server configurations, performance metrics, installed software, SQL Server and PostgreSQL database instances, and web apps (.NET on IIS and Java on Tomcat). Because no Azure connectivity is required, you can scan your estate locally and securely upload the data, saving time and avoiding complex networking or access approval requirements.

- Connectivity: Offline discovery
- Time to discovery: Quick to setup
- Assessment types: Lift and shift, modernization
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
    - After addressing issues, enable **incremental data** collection to retry only failed workloads
    - Export collected data
- Upload the collected data to an Azure Migrate project
- Enrich the discovery inventory
    - Apply custom tags such as department, business unit, scope
    - Apply reserved tags such as dev, test, retain, retire
- After the upload is successful, create reports, business cases and assessments
- Import additional inventory if needed

## Review before you start

- Set up the collector on a jump-box virtual machine (VM)
- Target environment:
    - vCenter
    - All versions of Windows and Linux
    - Web apps, database servers such as SQL Server, PostgreSQL and MySQL
- Prepare vCenter, guest and database accounts:
    - **vCenter account:** Read-only and guest operations (to collect server configuration and performance data from VMware machines)
    - **Windows:** Domain account or administrator* account (to collect installed software, SQL Server, PostgreSQL database instances and web apps configuration details)
    - **Linux:** Root account* (to collect installed software, SQL Server, PostgreSQL database instances and web apps configuration details)
    - **SQL Server:** Domain account with SQL permissions (to collect SQL readiness data)
    - **Note for \*:** You can configure least-privileged Windows, Linux, and SQL accounts by referring [this article](https://learn.microsoft.com/en-us/azure/migrate/best-practices-least-privileged-account?view=migrate).
- Azure Migrate Owner role is required to create an Azure Migrate Project
- Follow all [prerequisites](https://learn.microsoft.com/en-us/azure/migrate/how-to-discover-using-collector?view=migrate)
- Tag workloads correctly. This is very important for accurate assessments. Review details in the prerequisites document or [here](https://learn.microsoft.com/en-us/azure/migrate/assessment-prerequisites?view=migrate#tag-workloads-correctly).

## Create a new Azure Migrate project

- In the Azure portal, search for **Azure Migrate**.
- Under Services, select **Azure Migrate**.
- Under **Get started**, select **Create project**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-01.png)

- In **Create project**, select the Azure subscription and resource group. Create a resource group if you don't have one.
- Under **Project Details**, specify the project name, the geography in which you want to create the project and connectivity method. 
    - Review supported geographies for [public](https://learn.microsoft.com/en-us/azure/migrate/supported-geographies?view=migrate#public-cloud) and [government clouds](https://learn.microsoft.com/en-us/azure/migrate/supported-geographies?view=migrate#azure-government).
    - Create a project with [private endpoint connectivity](https://learn.microsoft.com/en-us/azure/migrate/discover-and-assess-using-private-endpoints?view=migrate#create-a-project-with-private-endpoint-connectivity).
    - Create a project in a [specific region](https://learn.microsoft.com/en-us/azure/migrate/quickstart-create-project?view=migrate#create-a-project-in-a-specific-region).
- Select **Create**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-02.png)

- Under **All projects**, review your **Azure Migrate Project**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-03.png)

## Download the Azure Migrate Collector

- In the Azure Migrate, select **All Projects -> your Azure Migrate Project**.
- In your Azure Migrate Project, select **Start discovery -> Using collector**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-04.png)

- In Discover, select **Download**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-05.png)

- Alternatively, you can download the **Azure Migrate collector installer** [here](https://aka.ms/Migrate/DownloadCollector). 

## Run the installer script on your jump-box

- Connect to your jump-box.
- Copy and extract the **AzureMigratecollector.zip** file to your jump-box.
- Launch PowerShell with administrative privileges.
- Change the directory to the extracted folder.
- Run the installer script:

    ```powershell
    .\AzureMigratecollector.ps1
    ```
- For the initial installation, select the **Fresh (F)** option. To upgrade the collector to a newer version, select the **Update (U)** option.

    ![Create Azure Migrate Project](/images/Azure-Migrate-06.png)

- The installer script performs the following actions:
    - Adds or updates registry keys at **HKLM:\Software\Microsoft\AzureAppliance**
    - Enables **IIS Role** and **other dependent features**
        - WAS, WAS-Process-Model, WAS-Config-APIs, Web-Server, Web-WebServer, Web-Mgmt-Service, Web-Request-Monitor, Web-Common-Http, Web-Static-Content, Web-Default-Doc, Web-Dir-Browsing, Web-Http-Errors, Web-App-Dev, Web-CGI, Web-Health, Web-Http-Logging, Web-Log-Libraries, Web-Security, Web-Filtering, Web-Performance, Web-Stat-Compression, Web-Mgmt-Tools, Web-Mgmt-Console, Web-Scripting-Tools, Web-Asp-Net45, Web-Net-Ext45, Web-Http-Redirect, Web-Windows-Auth, Web-Url-Auth
    - Creates files
        - **Config:** C:\ProgramData\Microsoft Azure\Config
        - **Offline Data:** C:\ProgramData\Microsoft Azure\OfflineData
    - Installs the following components
        - Microsoft Azure VMware Discovery Service.msi
        - Microsoft Azure VMware Assessment Service.msi
        - Microsoft Azure SQL Discovery and Assessment Service.msi
        - Microsoft Azure Web App Discovery and Assessment Service.msi
        - Microsoft Azure Server Discovery Service.msi
        - MicrosoftAzureApplianceConfigurationManager.msi
        - New Edge browser if not installed
    - Ensures that critical services for the Azure Migrate Appliance Configuration Manager are running
    - Launches the Azure Migrate Appliance Configuration Manager to start the onboarding process. You can also use the desktop shortcut to manually launch the **Azure Migrate Appliance Configuration Manager**.

        ![Create Azure Migrate Project](/images/Azure-Migrate-07.png)

## Collect data

The Azure Migrate Collector can be used to discover both VMware machines and physical servers in a hypervisor-agnostic manner. To collect data from physical servers, switch the fabric type at the top to **Physical**. Otherwise, keep it set to **VMware** to collect data from vCenter.

**Note:** After a fresh installation, all inputs may appear grayed out. You may not be able to switch the fabric type to **Physical**. Refresh the browser, and you will then be able to switch.

### Collect data from physical servers

- Switch the fabric type to **Physical**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-08.png)

- Provide credentials for discovery of Windows and Linux physical or virtual servers
    - Review supported [types of credentials](https://learn.microsoft.com/en-us/azure/migrate/add-server-credentials?view=migrate#types-of-server-credentials-supported)
    - Review the required permissions for [Windows Credentials](https://learn.microsoft.com/en-us/azure/migrate/tutorial-discover-physical?view=migrate-classic#prepare-windows-server) and [Linux Credentials](https://learn.microsoft.com/en-us/azure/migrate/tutorial-discover-physical?view=migrate-classic#prepare-linux-server)

    ![Create Azure Migrate Project](/images/Azure-Migrate-09.png)

- Provide physical and virtual server details
    - [Learn](https://learn.microsoft.com/en-us/azure/migrate/migrate-appliance?view=migrate#discovery-and-collection-process) more about the prerequisites for physical server discovery
    - You can add a single item, multiple items, or import a CSV file

    | Add single item | Add multiple items | Import CSV |
    |-----------------|-------------------|------------|
    | ![](../images/Azure-Migrate-10.png) | ![](../images/Azure-Migrate-11.png) | ![](../images/Azure-Migrate-12.png) |

    - To add servers using CSV import:
        - Download the CSV template
        - Add discovery source details to the CSV file. You can edit it using Microsoft Excel or a text editor such as Notepad. See the screenshots below.
        - Import and verify the CSV file
        - Save the discovery source

        ![Create Azure Migrate Project](/images/Azure-Migrate-13.png)

        ![Create Azure Migrate Project](/images/Azure-Migrate-14.png)

    - The collector communicates with: 
        - Windows servers using WinRM over port 5986 (HTTPS)
        - Linux servers using SSH over port 22 (TCP)
        - If HTTPS prerequisites aren't configured on Hyper‑V servers, the collector automatically switches to WinRM port 5985 (HTTP).
    - When you save, the collector validates connectivity to each server and displays the validation status in the table. If validation fails: 
        - Select **Validation failed** to review the error
        - Fix the issue
        - Validate again. 
        - You can revalidate connectivity at any time before starting data collection, or remove servers by selecting **Delete**.

        ![Create Azure Migrate Project](/images/Azure-Migrate-15.png)

- Provide credentials to collect SQL server instance readiness data
    - You can provide Windows, Domain accounts or Database specific user accounts for SQL Server. 
    - You can setup Database specific user accounts using an onboarding utility. [Learn more](https://learn.microsoft.com/en-us/azure/migrate/least-privilege-credentials?view=migrate).
    - Supported credential types include:
        - Domain credentials
        - Linux (Non-domain) credentials
        - Windows (Non-domain) credentials
        - SQL Server credentails for SQL Server Authentication

    ![Create Azure Migrate Project](/images/Azure-Migrate-16.png)

- Start data collection to collect inventory data

    ![Create Azure Migrate Project](/images/Azure-Migrate-17.png)

- Review collected data
    - Review the summary to understand the status and proportion of machines and worklaods that failed
    - Download the CSV file to review detailed error messages per server. Review the **Error ID** and **Error Message** columns in the CSV file. Columns in CSV file:
        - ServerName
        - WorkloadName
        - Category
        - Operating system
        - IPv6/IPv4
        - Source FQDN
        - Memory (MB)
        - Disks
        - Cores
        - Storage (GB)
        - Network adapters
        - MAC address
        - Boot type
        - Operating system type
        - Power status
        - Software
        - SQL instances
        - PGSQL instances
        - Web app
        - Severity
        - Message
        - Error ID
        - Error Message
        - Possible cause(s)
        - Recommended action(s)
        - Affected features    
    - Diagnose errors by fixing network access issues, modifying user privileges, or adding new credentials

    ![Create Azure Migrate Project](/images/Azure-Migrate-18.png)

- **Optional:** After resolving issues, run data collection again.
    - Enable **Collect incremental data only** to retry collecting data only for workloads where previous attempts failed. 
    - If **Collect incremental data only** is disabled, a full data collection will run.
    - Select **Start Data Collection** to run again

    ![Create Azure Migrate Project](/images/Azure-Migrate-19.png)

- Export collected data
    - (1) Select Export.
    - (2) The ZIP file will be saved in **C:\ProgramData\Microsoft Azure\OfflineData** location.

    ![Create Azure Migrate Project](/images/Azure-Migrate-20.png)

## Upload the collected data to an Azure Migrate project

Import the zip file generated by the collector

  - Open your **Azure Migrate Project** in the Azure portal
  - Select **Browse** and then choose the ZIP file exported from your collector.
  - After selecting the correct file, select **Import**.
  - You can monitor the import status as it progresses.
  - If you upload multiple files, the data will not be duplicated.

    ![Create Azure Migrate Project](/images/Azure-Migrate-21.png)

- Validate the import status

    ![Create Azure Migrate Project](/images/Azure-Migrate-22.png)

## Explore the inventory

- Review **All Inventory**

    ![Create Azure Migrate Project](/images/Azure-Migrate-23.png)

- It is recommended to set tags before running the collector. Review the prerequisites above for guidance. If tags were not set earlier, you can configure them here. There are custom and reserved tags. For example: 
    - **AzM_Environment:** If this tag is absent on the servers or workloads, they're considered as production workloads by default. If the workloads and servers operate in the dev/test environment, tag them with AzM.Environment: Dev.
    - **AzM_MigrationIntent:** To retain or retire workloads and servers, tag them with **AzM.MigrationIntent: Retain** or **AzM.MigrationIntent: Retire**. If the tag isn't applied, Azure Migrate treats the server or workload as a candidate for migration or modernization. 
    - You can export the inventory (**1 - Export Data -> All inventory**), update the **Environment** and **Migration Intent** columns in the CSV file, and then import the CSV file to update tags (**2 - Tags -> Import tags**)

    ![Create Azure Migrate Project](/images/Azure-Migrate-24.png)

- Explore other inventories in detail, such as:
    - Infrastructure
    - Databases
    - Web apps
    - Software
    - Insights

    ![Create Azure Migrate Project](/images/Azure-Migrate-25.png)

- Explore applications
    - Azure Migrate supports **automatic discovery of applications** by grouping inventory discovered using the Collector.
    - Currently, the process of automatically creating applications is performed only once after the inventory from collector is imported.
    - Each auto-discovered application represents a logical grouping of servers (and workloads running on those servers), automatically identified using server-naming patterns, inferred environments, and derived server roles.
    - Auto discovery of applications is currently only supported for collector-based inventory and not for appliance or CSV import based inventory.

    ![Create Azure Migrate Project](/images/Azure-Migrate-26.png)

## Generate Report

Azure Migrate reports provide a summarized view of migration and modernization opportunities in Azure. They include insights on workload readiness, security, and costs to help you prioritize workloads and make informed migration and modernization decisions. Reports can be generated after successful discovery and inventory enrichment.

**Note:** Report generation requires a business case, and a business case requires an assessment. Therefore, when you generate a report, Azure Migrate automatically creates the required assessment and business case.

- In your **Azure Migrate Project**, go to **Manage**, and then select **Reports**.
- Select **Generate Report**.
- Specify the report **name**.
- Select the report **type**. For example:
    - Azure Modernization and Migration Report
    - Security Insights Report
- Select the **Migration preference**. For example: 
    - **Modernize (AI ready):** Generates report with the preference to Modernize (AI ready) workloads to PaaS and make them AI Ready. In case, the workload cannot be modernized to PaaS, it is recommended for Lift and Shift migration to Azure VM.
    - **Migrate:** Generates report for a quick lift and shift migration of worklaods to Azure VM.
    - **Select** Modernize (AI ready).
- Select the **Configuration**. For example: 
    - **Define configuration:** Provide scope and settings to generate the report.
    - **Use configuration from an existing assessment:** Generate report using scope and settings from an existing assessment.
    - **Select** Define configuration. This creates a **Business case** and **Assessment**. This is the recommended approach.
- Next to **Report details**, select **Define configuration**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-27.png)

- Click **Add applications**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-28.png)

- Select the **application** and then click **Add**. If you select an application, you do not need to manually select its associated workloads, as all related workloads are included automatically.

    ![Create Azure Migrate Project](/images/Azure-Migrate-29.png)

- **Optional:** 
    - To include additional workloads that are not part of the selected application, select **Add workloads**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-30.png)

    Select **Add filter**, choose **Application name(s)** under Filter, select **Equals** under Operator, and then select all applications except those selected in the previous step. Click **Apply**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-31.png)

    Select the workload(s), and then click **Add selection**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-32.png)

- Click **Next**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-33.png)

- Work with your team to review and update the **General** settings, and then click **Next**. You can leave the default values if needed. These settings are used to calculate TCO and savings for the target Azure environment.
    - Target and pricing
        - **Default target location:** Specify the target Azure region to which you want to migrate your workloads. Target right-sizing and costing recommendations would be done based on the selected location.
        - **Default environment:** Specify the environment type for the workloads you intend to migrate. You can avail Azure discounts for Dev/Test workloads.
        - **Currency:** Specify the currency in which you would like to get your cost estimates.
        - **Program/offer:** Specify the Microsoft licensing program you would like to use for cost estimation. Select Enterprise Agreement support if you have a negotiated Enterprise agreement with Microsoft.
        - **Default savings option:** Optimize your cost estimates by selecting the applicable commitment based savings option.
        - **Discount(%):** Specify the additional discount that you are eligible apart from savings options.
        - **Subscription:** Select a subscription for cost estimation, only EA (Enterprise agreement) /MCA (Microsoft Customer Agreement) subscriptions are listed here.
        - **Uptime: Day(s) per month / Hour(s) per day** Specify the time for which you expect the workloads to run.
    - Assessment criteria
        - **Sizing criteria:** Performance-based sizing considers the resource utilization and configuration attributes of workloads to right-sizes the targets accordingly for Azure. As on-premises sizing considers only configuration attributes of the on-premises workloads.
        - **Performance history:** The duration of performance history you would like to consider for the on-premises workloads.
        - **Percentile utilization:** The percentile value that you would like to consider for the performance history of the on-premises workloads.
        - **Comfort factor:** Comfort factor is a buffer that is added on top of utilization to account for situations like seasonal spikes in usage, insufficient performance data, likely increase in future usage, etc. [Learn more about comfort factor](https://go.microsoft.com/fwlink/?linkid=2342615).
    - Azure hybrid benefit: Apply Azure hybrid benefit and save up to 85% vs. pay-as-you-go rate by bringing your Windows Server licenses to Azure. [Review Azure hybrid benefit compliance](https://go.microsoft.com/fwlink/?LinkId=859786).
        -  **I have a Windows Server license:** Azure Hybrid Benefit allows Microsoft customers with Windows Server Software Assurance or Windows Server subscriptions to bring their licenses to Azure. [Learn more](https://go.microsoft.com/fwlink/?linkid=2257523)
        -  **I have Enterprise Linux subscriptions:** Azure Hybrid Benefit allows Microsoft customers with existing Enterprise Linux subscription to bring them to Azure. [Learn more](https://go.microsoft.com/fwlink/?linkid=2257523)
   -  Security
         -  **Include Microsoft Defender for cloud?:** Specify if you wish to include Defender for Server cost (Plan 2) with Microsoft Defender for cloud to protect your servers in Azure.

    ![Create Azure Migrate Project](/images/Azure-Migrate-34.png)

- Work with your team to review and update the **Advanced** settings, and then click **Save**.
    - Infrastructure settings
    - Database settings
    - Web app settings
    - Application settings

    ![Create Azure Migrate Project](/images/Azure-Migrate-35.png)

- Click **Generate Report**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-36.png)

- Report generation creates the following artifacts. You can download and review the report.
    - (1) Assessment
    - (2) Business case
    - (3) Report

    ![Create Azure Migrate Project](/images/Azure-Migrate-37.png)

## Next steps

- [Review the report](azure-migrate-view-report.md)
- Review the business case
- Review the assessment