# Review Azure Arc-enabled environment

This guide explains how to create an Azure Arc inventory report.

---

## Prerequisites
- An active Azure subscription.
- Permissions to create and edit Workbooks in the target subscription/resource group.
- Access to Azure Resource Graph (required for queries).
- Azure Arc inventory workbook.

## Steps to Import the Workbook

- Download the [Galery Template Workbook file](../Workbooks/Azure%20Arc%20Inventory.workbook) from this repository.
- Sign in to the [Azure Portal](https://portal.azure.com).
- In the search bar, type **Workbooks** and open the service.
- Click **New** to create a new Workbook.
- From the toolbar, select **Advanced Editor**.
- In the editor, choose **Gallery template**.
- Paste the workbook content from this repository into the editor.
- Click **Apply**.
- Save the Workbook into your desired **Subscription**, **Resource Group**, and **Location**.


