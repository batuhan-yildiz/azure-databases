# Azure Environment Overview

This page provides a high-level view of the current Azure Lab environment including virtual networks, workloads, and VNet peering configuration.

Use it to understand the logical topology before deploying or extending additional workloads.

## Virtual Networks

| Resource Group  | VNet Name       | Address Space | Subnet Name       | Subnet Prefix | Purpose                                               |
| --------------- | --------------- | ------------- | ----------------- | ------------- | ----------------------------------------------------- |
| `rg-hub`        | `vnet-hub`      | `10.0.0.0/16` | `subnet-hub`      | `10.0.0.0/24` | Shared resources (DNS, monitoring, private endpoints) |
| `rg-jumpbox`    | `vnet-jumpbox`  | `10.1.0.0/16` | `subnet-jumpbox`  | `10.1.0.0/24` | Jumpbox virtual machine and admin access point        |
| `rg-oracle01`   | `vnet-oracle01` | `10.5.0.0/16` | `subnet-oracle01` | `10.5.0.0/24` | Oracle virtual machine        |

## Workloads

| Resource Name | Resource Group | VNet           | Subnet           | Public IP                   | Private IP | Purpose                                                  |
| ------------- | -------------- | -------------- | ---------------- | --------------------------- | ---------- | -------------------------------------------------------- |
| `JumpboxVM01` | `rg-jumpbox`   | `vnet-jumpbox` | `subnet-jumpbox` | `<your-assigned-public-ip>` | `10.1.0.4` | Secure admin VM used for migrations and management tasks |

## VNet Peering Configuration

| Peering Name     | Source VNet    | Remote VNet    | Direction |
| ---------------- | -------------- | -------------- | --------- |
| `jumpbox-to-hub` | `vnet-jumpbox` | `vnet-hub`     | Outbound  |
| `hub-to-jumpbox` | `vnet-hub`     | `vnet-jumpbox` | Inbound   |
