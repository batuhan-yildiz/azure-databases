# Azure Environment Overview

This page provides a high-level view of the current Azure Lab environment including virtual networks, workloads, and VNet peering configuration.

Use it to understand the logical topology before deploying or extending additional workloads.

## Workloads

| Resource Name       | Resource Group  | VNet            | Subnet                         | Public IP              | Private IP | Purpose                                       |
| ------------------- | --------------- | --------------- | ------------------------------ | ---------------------- | ---------- | --------------------------------------------- |
| `JumpboxVM01`       | `rg-jumpbox`    | `vnet-jumpbox`  | `subnet-vnet-jumpbox-backend`  | `<assigned-public-ip>` | `10.1.0.4` | Jumpbox VM                                    |
| `OracleVM01`        | `rg-oracle01`   | `vnet-oracle01` | `subnet-vnet-oracle01-backend` | `No Public IP`         | `10.5.0.4` | Oracle-Linux 8.10 and Oracle Database 21c XE  |
| `azpgsqlprod01`     | `rg-pgsql03`    | `vnet-pgsql03`  | `subnet-vnet-pgsql03-backend`  | `No Public IP`         | `10.6.0.4` | Azure Database for PostgreSQL Flexible Server |
| `AzOpenAIModern01 ` | `rg-openai01`   | `vnet-openai01` | `subnet-vnet-openai01-backend` | `No Public IP`         | `10.7.0.4` | Azure OpenAI with gpt-4.1 model               |


## Private DNS Zones

| Private DNS Zones                         | Resource Group  |
| ----------------------------------------- | --------------- |
| `privatelink.postgres.database.azure.com` | `rg-hub`        |
| `privatelink.openai.azure.com`            | `rg-hub`        |

## Private DNS Links

| Virtual Network | Private DNS Link Name             | Private DNS Zones                         | Resource Group  |
| --------------- | --------------------------------- | ----------------------------------------- | --------------- |
| `vnet-jumpbox`  | `link-to-vnet-jumpbox-postgresql` | `privatelink.postgres.database.azure.com` | `rg-hub`        |
| `vnet-jumpbox`  | `link-to-vnet-jumpbox-openai`     | `privatelink.openai.azure.com`            | `rg-hub`        |
| `vnet-pgsql03`  | `link-to-vnet-pgsql03-postgresql` | `privatelink.postgres.database.azure.com` | `rg-hub`        |
| `vnet-openai01` | `link-to-vnet-openai01-openai`    | `privatelink.openai.azure.com`            | `rg-hub`        |


## Virtual Networks

| Resource Group   | VNet Name        | Address Space | Subnet Name                    | Subnet Prefix |
| ---------------- | ---------------- | ------------- | ------------------------------ | ------------- |
| `rg-hub`         | `vnet-hub`       | `10.0.0.0/16` | `subnet-vnet-hub-backend`      | `10.0.0.0/24` |
| `rg-jumpbox`     | `vnet-jumpbox`   | `10.1.0.0/16` | `subnet-vnet-jumpbox-backend`  | `10.1.0.0/24` |
| `rg-oracle01`    | `vnet-oracle01`  | `10.5.0.0/16` | `subnet-vnet-oracle01-backend` | `10.5.0.0/24` |
| `rg-pgsql03`     | `vnet-pgsql03`   | `10.6.0.0/16` | `subnet-vnet-pgsql03-backend`  | `10.6.0.0/24` |
| `rg-openai01`    | `vnet-openai01`  | `10.6.0.0/16` | `subnet-vnet-openai01-backend` | `10.6.0.0/24` |


## VNet Peering Configuration

| Peering Name          | Source VNet       | Remote VNet      | Direction |
| --------------------- | ----------------- | ---------------  | --------- |
| `jumpbox-to-hub`      | `vnet-jumpbox`    | `vnet-hub`       | Outbound  |
| `hub-to-jumpbox`      | `vnet-hub`        | `vnet-jumpbox`   | Inbound   |
| `oracle01-to-hub`     | `vnet-oracle01`   | `vnet-hub`       | Outbound  |
| `hub-to-oracle01`     | `vnet-hub`        | `vnet-oracle01`  | Inbound   |
| `oracle01-to-jumpbox` | `vnet-oracle01`   | `vnet-jumpbox`   | Outbound  |
| `jumpbox-to-oracle01` | `vnet-jumpbox`    | `vnet-oracle01`  | Inbound   |
| `pgsql03-to-hub`      | `vnet-pgsql03`    | `vnet-hub`       | Outbound  |
| `hub-to-pgsql03`      | `vnet-hub`        | `vnet-pgsql03`   | Inbound   |
| `pgsql03-to-jumpbox`  | `vnet-pgsql03`    | `vnet-jumpbox`   | Outbound  |
| `jumpbox-to-pgsql03`  | `vnet-jumpbox`    | `vnet-pgsql03`   | Inbound   |
| `openai01-to-hub`     | `vnet-openai01`   | `vnet-hub`       | Outbound  |
| `hub-to-openai01`     | `vnet-hub`        | `vnet-openai01`  | Inbound   |
| `openai01-to-jumpbox` | `vnet-openai01`   | `vnet-jumpbox`   | Outbound  |
| `jumpbox-to-openai01` | `vnet-jumpbox`    | `vnet-openai01`  | Inbound   |
