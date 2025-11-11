# Azure Environment Overview

This page provides a high-level view of the current Azure Lab environment including virtual networks, workloads, and VNet peering configuration.

Use it to understand the logical topology before deploying or extending additional workloads.

## Virtual Networks

| Resource Group  | VNet Name       | Address Space | Subnet Name       | Subnet Prefix | Purpose                                               |
| --------------- | --------------- | ------------- | ----------------- | ------------- | ----------------------------------------------------- |
| `rg-hub`        | `vnet-hub`      | `10.0.0.0/16` | `subnet-hub`      | `10.0.0.0/24` | Shared resources (DNS, monitoring, private endpoints) |
| `rg-jumpbox`    | `vnet-jumpbox`  | `10.1.0.0/16` | `subnet-jumpbox`  | `10.1.0.0/24` | Jumpbox virtual machine and admin access point        |
| `rg-oracle01`   | `vnet-oracle01` | `10.5.0.0/16` | `subnet-oracle01` | `10.5.0.0/24` | Oracle to PostgreSQL Migration                        |
| `rg-pgsql03`    | `vnet-pgsql03`  | `10.6.0.0/16` | `subnet-pgsql03`  | `10.6.0.0/24` | Oracle to PostgreSQL Migration                        |

## Workloads

| Resource Name    | Resource Group  | VNet            | Subnet            | Public IP                   | Private IP | Purpose                                                  |
| ---------------- | --------------- | --------------- | ----------------- | --------------------------- | ---------- | -------------------------------------------------------- |
| `JumpboxVM01`    | `rg-jumpbox`    | `vnet-jumpbox`  | `subnet-jumpbox`  | `<your-assigned-public-ip>` | `10.1.0.4` | Secure admin VM used for migrations and management tasks |
| `OracleVM01`     | `rg-oracle01`   | `vnet-oracle01` | `subnet-oracle01` | `<your-assigned-public-ip>` | `10.5.0.4` | Oracle to PostgreSQL Migration tasks                      |
| `azpgsqlprod01`  | `rg-pgsql03`    | `vnet-pgsql03`  | `subnet-pgsql03`  | No Public IP                | `10.6.0.4` | Oracle to PostgreSQL Migration tasks                      |

## VNet Peering Configuration

| Peering Name          | Source VNet     | Remote VNet     | Direction |
| --------------------- | --------------- | --------------- | --------- |
| `jumpbox-to-hub`      | `vnet-jumpbox`  | `vnet-hub`      | Outbound  |
| `hub-to-jumpbox`      | `vnet-hub`      | `vnet-jumpbox`  | Inbound   |
| `oracle01-to-hub`     | `vnet-oracle01` | `vnet-hub`      | Outbound  |
| `hub-to-oracle01`     | `vnet-hub`      | `vnet-oracle01` | Inbound   |
| `oracle01-to-jumpbox` | `vnet-oracle01` | `vnet-jumpbox`  | Outbound  |
| `jumpbox-to-oracle01` | `vnet-jumpbox`  | `vnet-oracle01` | Inbound   |
| `pgsql03-to-hub`     | `vnet-pgsql03`   | `vnet-hub`      | Outbound  |
| `hub-to-pgsql03`     | `vnet-hub`       | `vnet-pgsql03`  | Inbound   |
| `pgsql03-to-jumpbox` | `vnet-pgsql03`   | `vnet-jumpbox`  | Outbound  |
| `jumpbox-to-pgsql03` | `vnet-jumpbox`   | `vnet-pgsql03`  | Inbound   |
