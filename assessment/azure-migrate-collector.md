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
- For the first installation, select the fresh (F) option. To upgrade the collector to a newer version, select the update (U) option.

    ![Create Azure Migrate Project](/images/Azure-Migrate-06.png)

- The installer script performs the following actions:
    - Adding\Updating Registry Keys at HKLM:\Software\Microsoft\AzureAppliance
    - Enabling IIS Role and other dependent features
        - WAS, WAS-Process-Model, WAS-Config-APIs, Web-Server, Web-WebServer, Web-Mgmt-Service, Web-Request-Monitor, Web-Common-Http, Web-Static-Content, Web-Default-Doc, Web-Dir-Browsing, Web-Http-Errors, Web-App-Dev, Web-CGI, Web-Health, Web-Http-Logging, Web-Log-Libraries, Web-Security, Web-Filtering, Web-Performance, Web-Stat-Compression, Web-Mgmt-Tools, Web-Mgmt-Console, Web-Scripting-Tools, Web-Asp-Net45, Web-Net-Ext45, Web-Http-Redirect, Web-Windows-Auth, Web-Url-Auth
    - Creating files
        - Config: C:\ProgramData\Microsoft Azure\Config
        - Offline Data: C:\ProgramData\Microsoft Azure\OfflineData
    - Installing
        - Microsoft Azure VMware Discovery Service.msi
        - Microsoft Azure VMware Assessment Service.msi
        - Microsoft Azure SQL Discovery and Assessment Service.msi
        - Microsoft Azure Web App Discovery and Assessment Service.msi
        - Microsoft Azure Server Discovery Service.msi
        - MicrosoftAzureApplianceConfigurationManager.msi
        - New Edge browser if not installed
    - Ensuring critical services for Azure Migrate appliance configuration manager are running
    - Launching Azure Migrate appliance configuration manager to start the onboarding process. You may use the shortcut placed on the desktop to manually launch **Azure Migrate appliance configuration manager**.

        ![Create Azure Migrate Project](/images/Azure-Migrate-07.png)

## Collect data

The same Azure migrate collector can be used to discover both VMware machines and physical servers that’s hypervisor agnostic. To collect data about physical servers, switch the fabric type at the top to physical. Otherwise keep it as VMWare to collect data from vCenter.

**Note:** After fresh install, all inputs may be grayed out. You may not e able to switch Fabric Type to Physical. You need to refresh the browser and then you will be able to switch.

### Collect data from physical servers

- Switch Fabric Type to **Physical**.

    ![Create Azure Migrate Project](/images/Azure-Migrate-08.png)

- Provide credentials for discovery of Windows and Linux physical or virtual servers
    - Review supported [types of credentials](https://learn.microsoft.com/en-us/azure/migrate/add-server-credentials?view=migrate#types-of-server-credentials-supported)
    - Review the required permissions for [Windows Credentials](https://learn.microsoft.com/en-us/azure/migrate/tutorial-discover-physical?view=migrate-classic#prepare-windows-server) and [Linux Credentials](https://learn.microsoft.com/en-us/azure/migrate/tutorial-discover-physical?view=migrate-classic#prepare-linux-server)

    ![Create Azure Migrate Project](/images/Azure-Migrate-09.png)

- Provide physical and virtual server details
    - [Learn](https://learn.microsoft.com/en-us/azure/migrate/migrate-appliance?view=migrate#discovery-and-collection-process) more about the prerequisites for physical server discovery
    - You can add a single item, or multiple items, or import CSV


    <table>
    <tr>
        <th>Add single item</th>
        <th>Add multiple items</th>
        <th>Import CSV</th>
    </tr>
    <tr>
        <td style="vertical-align: top;">
        <img src="../images/Azure-Migrate-10.png" width="300">
        </td>
        <td style="vertical-align: top;">
        <img src="../images/Azure-Migrate-11.png" width="300">
        </td>
        <td style="vertical-align: top;">
        <img src="../images/Azure-Migrate-12.png" width="300">
        </td>
    </tr>
    </table>


