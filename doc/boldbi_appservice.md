# How to Deploy Bold BI on Azure App Services (Linux-based Container)

This document provides a step-by-step guide to deploy **Bold BI** on **Azure App Services** using a **Linux-based container** via an ARM (Azure Resource Manager) template.

---

## Prerequisites

1. An active **Azure subscription** with **Contributor** access.
2. A valid **Bold BI license** (optional for trial deployment).
3. A supported **database server**.
   - Microsoft SQL Server 2016+
   - PostgreSQL 13.0+
   - MySQL 8.0+
   - Oracle Database 19c+
4. A modern web browser to access the **Azure Portal**.
   - Microsoft Edge
   - Mozilla Firefox
   - Chrome

---

## Deployment Steps

### Step 1: Sign in to the Azure Portal

1. Open your browser and navigate to the [Azure Portal](https://portal.azure.com).
2. Log in with your Azure account credentials.

---

### Step 2: Open the Custom Template Deployment

1. In the Azure Portal **search bar**, type **`Deploy a custom template`**.
2. From the results, select **Deploy a custom template** (under *Marketplace* or *Services*).

    ![deploy_a_custom_template](../images/custom_template.png)

---

### Step 3: Build Your Own Template

1. On the deployment page, choose the **Build your own template in the editor** option.
  ![build_your_own_template](../images/own_template.png)

2. Open the [ARM template JSON file](../arm-template/boldbi-appservice.json), copy the entire JSON content, and paste it into the editor.
  ![build_your_own_template_edit](../images/own_template_save.png)

3. Click **Save + Continue**.

---

### Step 4: Fill in Deployment Details

Provide the following parameters:

#### Mandatory (always required)

| Parameter | Description |
|------------|-------------|
| **Subscription** | Choose your Azure subscription. |
| **Resource group** | Select an existing resource group or create a new one. |
| **Region** | Choose the Azure region for deployment. |
| **App Service Name** | Enter a unique name for the Bold BI App URL (3–24 characters, lowercase letters and numbers only). If taken, the deployment fails — choose another. |
| **App Service Plan Size** | Select the App Service SKU. Available values: <br>• `P1V3_2Core_8GB_DEV` <br>• `P2V3_4Core_16GB_PROD` <br>• `P3V3_8Core_32GB_PROD` |
| **Storage Account Name** | Unique name (3–24 characters, lowercase letters and numbers only) for Blob storage. |

  ![mandatory](../images/mandatory_value.png)

#### Required for Auto-Deployment Mode

> Provide these parameters to enable full automatic configuration of database connection and admin user (no manual steps required after deployment).

| Parameter | Description |
|------------|-------------|
| **User Email** | Initial administrator email address. |
| **User Password** | Admin password. The password must meet the following requirements: <br>• At least **6 characters** <br>• Includes **1 uppercase** letter <br>• Includes **1 lowercase** letter <br>• Includes **1 numeric** character <br>• Includes **1 special** character |

#### Database Configuration

| Parameter | Description |
|------------|-------------|
| **Database Server Type** | Choose one: `mssql`, `postgresql`, `mysql`, or `oracle`. |
| **Database Host** | Database server hostname or endpoint. |
| **Database User** | Database username. |
| **Database Password** | Database password. |

#### Optional / Additional Database Parameters

> Only required in specific cases.

| Parameter | Description |
|------------|-------------|
| **Database Port** | Database port (optional; defaults based on server type). |
| **Postgres Maintenance Database** | For PostgreSQL servers. The system uses `postgres` by default. Provide a different value if your server uses a different default database. |
| **Database Name** | Existing database name. If omitted, Bold BI creates `bold_services`. |
| **Database Additional Parameters** | Additional connection string parameters if required. See official docs: <br>• [MySQL](https://dev.mysql.com/doc/connector-net/en/connector-net-8-0-connection-options.html) <br>• [PostgreSQL](https://www.npgsql.org/doc/connection-string-parameters.html) <br>• [MS SQL](https://learn.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlconnection.connectionstring?view=netframework-4.8.1) <br>• [Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/19/odpnt/ConnectionConnectionString.html) |

#### Optional Licensing & Activation

> Not required for deployment; defaults allow trial or manual activation post-deployment.

| Parameter | Description |
|------------|-------------|
| **Unlock Key** | Bold BI license key. |

---

### Step 5: Review and Create

1. After all details are filled in, click **Review + Create**.
  ![review_and_save](../images/review_create.png)

2. Wait for validation to complete.

3. Click **Create** to begin the deployment.

---

### Step 6: Wait for Resources to Be Created

1. Once the deployment finishes, click **Go to resource group**.
  ![resource_group](../images/resource_group.png)

2. From the resource group, select the **App Service**.
  ![app_service](../images/app_service.png)
---

### Step 7: Wait for the Site to Be Healthy

1. Wait approximately **10 minutes** for the site to come up and run.
2. On the **Overview** page, check the **Health Check** status:
   - `Health Check: 100.00% (Healthy 1 / Degraded 0)` indicates the site is healthy.
   ![health_check](../images/health_check.png)

---

### Step 8: Access the Bold BI Application

1. On the **Overview** page, copy the **Default Domain** URL.
  ![default_domain](../images/default_domain.png)

2. Paste the URL into your browser.
---

### Step 9: Start the Bold BI Application

After accessing the Bold BI site, follow the application startup guide: [https://help.boldbi.com/application-startup/](https://help.boldbi.com/application-startup/)

---

## How to Install Client Libraries for Bold BI in Azure App Service

To install additional **client libraries** (e.g., MongoDB, ClickHouse, Google BigQuery, etc.) for Bold BI:

1. Navigate to **App Service** → **Settings** → **Environment variables**.
2. Click the **+ Add** button to add a new environment variable.
  ![environment_variables](../images/add_new_environment.png)

3. Set the environment variable as follows:
   - **Name:** `OPTIONAL_LIBS`
   - **Value:** `mongodb,mysql,influxdb,snowflake,oracle,clickhouse,google`
  ![environment_variables](../images/update_client_libraries_value.png)

   - **Default client libraries included in Bold BI:**
    `mongodb, mysql, influxdb, snowflake, oracle, clickhouse, google` For more details, refer to the official documentation: [Consent to Deploy Client Libraries](https://github.com/boldbi/boldbi-server-in-docker/blob/main/docs/consent-to-deploy-client-libraries.md)

4. Click **Apply** / **Save** to apply the changes.
5. Restart the App Service for the new libraries to take effect.

---

## How to Upgrade Bold BI Docker Images

To upgrade the Bold BI version running on your Azure App Service:

1. Navigate to **App Service** → **Deployment** → **Deployment Center** → **Container**.
2. In the **Image and tag** field, update the value with the new Bold BI image name and tag.
3. Click **Save** at the top of the page.
  ![upgrade_image](../images/update_new_docker_image.png)

4. Wait approximately **10 minutes** for the new Docker image to be pulled and the site to come up.
5. To monitor the upgrade progress, Navigate to **App Service** → **Deployment** → **Deployment Center** → **Logs**.
  ![upgrade_logs](../images/update_new_docker_image_logs.png)

---

## Troubleshooting

### 1. Health Check Not Reaching 100%

If the health check is not up and running at 100% after 10 minutes:

1. Navigate to **App Service** → **Deployment** → **Deployment Center** → **Logs**.
2. Review the **platform-level logs** to identify the issue (image pull failures, container startup errors, etc.).
  ![upgrade_logs](../images/update_new_docker_image_logs.png)

---

### 2. Application-Level Issues

If you encounter issues at the **Bold BI application level**:

1. Navigate to **App Service** → **Monitoring** → **Logs**.
2. Under **Tables**, select `AppServiceConsoleLogs`.
  ![view_logs](../images/view_logs.png)
  
3. Review the application-level logs to diagnose the problem.
  ![show_logs](../images/show_logs.png)
---

## Quick Reference: Useful Links

| Resource | Link |
|----------|------|
| Azure Portal | [https://portal.azure.com](https://portal.azure.com) |
| Bold BI Application Startup | [https://help.boldbi.com/application-startup/](https://help.boldbi.com/application-startup/) |
| Client Libraries Documentation | [GitHub - boldbi-server-in-docker](https://github.com/boldbi/boldbi-server-in-docker/blob/main/docs/consent-to-deploy-client-libraries.md) |
| MS SQL Connection String | [Microsoft Docs](https://learn.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlconnection.connectionstring?view=netframework-4.8.1) |
| PostgreSQL Connection String | [Npgsql Docs](https://www.npgsql.org/doc/connection-string-parameters.html) |
| MySQL Connection Options | [MySQL Docs](https://dev.mysql.com/doc/connector-net/en/connector-net-8-0-connection-options.html) |
| Oracle Connection String | [Oracle Docs](https://docs.oracle.com/en/database/oracle/oracle-database/19/odpnt/ConnectionConnectionString.html) |

---

## Summary

This guide covers the complete workflow to:

- Deploy **Bold BI** on **Azure App Services** using a Linux container.
- Install **client libraries** via environment variables.
- Upgrade the Bold BI Docker image when a new version is available.
- Troubleshoot common deployment and runtime issues.
