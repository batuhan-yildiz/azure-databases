# Azure Environment Overview

This page provides a high-level view of the current Azure Lab environment including virtual networks, workloads, and VNet peering configuration.

Use it to understand the logical topology before deploying or extending additional workloads.

## Azure Environment

![Azure Environment](../images/Environment-Azure.png)

![Onprem Environment](../images/Environment-OnPrem.png)


## Workloads

| Resource Name       | Resource Group  | VNet            | Public IP              | Private IP | Description                                   |
| ------------------- | --------------- | --------------- | ---------------------- | ---------- | --------------------------------------------- |
| `JumpboxVM01`       | `rg-jumpbox`    | `vnet-jumpbox`  | `<assigned-public-ip>` | `10.1.0.4` | Jumpbox VM                                    |
| `HyperHostVM01`     | `rg-onprem`     | `vnet-onprem`   | `<assigned-public-ip>` | `10.2.0.4` | HyperV VM with nested VMs                     |
| `azsqlmidevtest01`  | `rg-sqlmi01`    | `vnet-sqlmi01`  | `No Public IP`         | `10.3.0.9` | Azure SQL Managed Instance                    |
| `OracleVM01`        | `rg-oracle01`   | `vnet-oracle01` | `No Public IP`         | `10.4.0.4` | Oracle-Linux 8.10 and Oracle Database 21c XE  |
| `azpgsqlprod01`     | `rg-pgsql03`    | `vnet-pgsql03`  | `No Public IP`         | `10.5.0.4` | Azure Database for PostgreSQL Flexible Server |
| `AzOpenAIModern01 ` | `rg-openai01`   | `vnet-openai01` | `No Public IP`         | `10.6.0.4` | Azure OpenAI with gpt-4.1 model               |


## Private DNS Zones

| Private DNS Zones                         | Resource Group  |
| ----------------------------------------- | --------------- |
| `privatelink.postgres.database.azure.com` | `rg-hub`        |
| `privatelink.openai.azure.com`            | `rg-hub`        |
| `privatelink.database.windows.net`        | `rg-hub`        |

## Private DNS Links

| Virtual Network | Private DNS Link Name             | Private DNS Zones                         |
| --------------- | --------------------------------- | ----------------------------------------- |
| `vnet-jumpbox`  | `link-to-vnet-jumpbox-postgresql` | `privatelink.postgres.database.azure.com` |
| `vnet-jumpbox`  | `link-to-vnet-jumpbox-openai`     | `privatelink.openai.azure.com`            |
| `vnet-jumpbox`  | `link-to-vnet-jumpbox-azuresql`   | `privatelink.database.windows.net`        |
| `vnet-pgsql01`  | `link-to-vnet-pgsql01-postgresql` | `privatelink.postgres.database.azure.com` |
| `vnet-openai01` | `link-to-vnet-openai01-openai`    | `privatelink.openai.azure.com`            |
| `vnet-sqlmi01`  | `link-to-vnet-sqlmi01-azuresqlmi` | `privatelink.database.windows.net`        |



