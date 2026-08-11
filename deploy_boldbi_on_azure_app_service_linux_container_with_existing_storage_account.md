# How to Deploy Bold BI on Azure App Services (Linux Container) with an Existing Storage Account

This guide explains how to deploy **Bold BI** on **Azure App Services (Linux container)** while reusing the **existing Azure Storage Account** that was originally used with the **Windows-based App Service**.

> If you previously hosted Bold BI on an Azure App Service for Windows, you can keep using the same storage account — the Linux container will connect to it using the same connection details.

---

## Prerequisites

1. You must already have a working **Bold BI deployment on Azure App Service (Windows)**. If you have not deployed it yet, follow the Azure App Service [deployment guide](https://help.boldbi.com/deploying-bold-bi/deploying-on-azure-app-service/)

   - Before creating the new **Linux Container-based Azure App Service**, stop the existing Windows-based Azure App Service to avoid conflicts and unnecessary resource usage.

   - After the Linux Container deployment is completed and your Bold BI site is verified to be working correctly, you can either keep the Windows-based App Service as a backup or delete it if it is no longer required.
2. Collect the details from the **old storage account**:
   - Storage account name
   - Storage account access key
   - Azure Blob container name
4. You have an **Azure subscription** with permission to deploy resources, and an existing **resource group** to deploy into.

---

## Deployment Steps

### Step 1: Sign in to the Azure Portal

1. Open your browser and navigate to the [Azure Portal](https://portal.azure.com).
2. Log in with your Azure account credentials.

---

### Step 2: Open the Custom Template Deployment

1. In the Azure Portal **search bar**, type **`Deploy a custom template`**.
2. From the results, select **Deploy a custom template** (under *Marketplace* or *Services*).

    ![deploy_a_custom_template](./images/custom_template.png)

---

### Step 3: Build Your Own Template

1. On the deployment page, choose the **Build your own template in the editor** option.
   ![build_your_own_template](./images/own_template.png)

2. Click the **▶ Click here to expand the ARM template** section below, then copy the JSON and paste it into the editor.
   ![build_your_own_template_edit](./images/own_template_save.png)

3. Click **Save + Continue**.

<details>
<summary>Click here to expand the ARM template</summary>

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appServiceName": {
      "type": "string",
      "maxLength": 24,
      "minLength": 3
    },
    "storageAccountName": {
      "type": "string",
      "metadata": {
        "description": "Existing storage account name (reused from the previous Windows-based App Service)."
      }
    },
    "storageAccountAccessKey": {
      "type": "secureString",
      "metadata": {
        "description": "Existing storage account access key. Stored as the BOLD_SERVICES_AZUREBLOB_ACCESSKEY app setting."
      }
    },
    "blobContainerName": {
      "type": "string",
      "metadata": {
        "description": "Existing blob container name in the storage account."
      }
    },
    "imageName": {
      "type": "string",
      "defaultValue": "asme123/boldbi:azure_app_services_images4"
    },
    "protocol": {
      "type": "string",
      "defaultValue": "https",
      "allowedValues": [
        "http",
        "https"
      ]
    },
    "appServicePlanSize": {
      "type": "string",
      "defaultValue": "P1V3_2Core_8GB_DEV",
      "allowedValues": [
        "P1V3_2Core_8GB_DEV",
        "P2V3_4Core_16GB_PROD",
        "P3V3_8Core_32GB_PROD"
      ]
    }
  },
  "variables": {
    "planName": "[concat(parameters('appServiceName'), '-plan')]",
    "workspaceName": "[concat(parameters('appServiceName'), '-logs')]",
    "httpsOnly": "[equals(parameters('protocol'), 'https')]",
    "azureAppServicesHttps": "[if(equals(parameters('protocol'), 'https'), 'true', 'false')]",
    "planSkuMap": {
      "P1V3_2Core_8GB_DEV": "P1v3",
      "P2V3_4Core_16GB_PROD": "P2v3",
      "P3V3_8Core_32GB_PROD": "P3v3"
    },
    "skuName": "[variables('planSkuMap')[parameters('appServicePlanSize')]]",
    "dockerImage": "[concat('DOCKER|', parameters('imageName'))]",
    "blobStorageUri": "[concat(parameters('storageAccountName'), '.blob.core.windows.net')]"
  },
  "resources": [
    {
      "type": "Microsoft.OperationalInsights/workspaces",
      "apiVersion": "2023-09-01",
      "name": "[variables('workspaceName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "sku": {
          "name": "PerGB2018"
        },
        "retentionInDays": 30
      }
    },
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2023-12-01",
      "name": "[variables('planName')]",
      "location": "[resourceGroup().location]",
      "kind": "linux",
      "sku": {
        "name": "[variables('skuName')]",
        "tier": "PremiumV3",
        "capacity": 1
      },
      "properties": {
        "reserved": true
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2023-12-01",
      "name": "[parameters('appServiceName')]",
      "location": "[resourceGroup().location]",
      "kind": "app,linux,container",
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('planName'))]"
      ],
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('planName'))]",
        "httpsOnly": "[variables('httpsOnly')]",
        "siteConfig": {
          "linuxFxVersion": "[variables('dockerImage')]",
          "alwaysOn": true,
          "healthCheckPath": "/api/status",
          "healthCheckEvictionTimeInMin": 2,
          "appSettings": [
            {
              "name": "AZURE_APP_SERVICES_HTTPS",
              "value": "[variables('azureAppServicesHttps')]"
            },
            {
              "name": "APP_URL",
              "value": "[concat(parameters('protocol'), '://', parameters('appServiceName'), '.azurewebsites.net')]"
            },
            {
              "name": "WEBSITES_PORT",
              "value": "80"
            },
            {
              "name": "WEBSITES_ENABLE_APP_SERVICE_STORAGE",
              "value": "true"
            },
            {
              "name": "BOLD_SERVICES_AZUREBLOB_ACCESSKEY",
              "value": "[parameters('storageAccountAccessKey')]"
            },
            {
              "name": "BOLD_SERVICES_AZUREBLOB_ACCOUNT_NAME",
              "value": "[parameters('storageAccountName')]"
            },
            {
              "name": "BOLD_SERVICES_AZUREBLOB_CONNECTION_TYPE",
              "value": "https"
            },
            {
              "name": "BOLD_SERVICES_AZUREBLOB_STORAGE_URI",
              "value": "[variables('blobStorageUri')]"
            },
            {
              "name": "BOLD_SERVICES_STORAGETYPE",
              "value": "1"
            },
            {
              "name": "BOLD_SERVICES_AZUREBLOB_CONTAINER_NAME",
              "value": "[parameters('blobContainerName')]"
            },
            {
              "name": "BOLD_SERVICES_CONSOLE_LOG_ENABLED",
              "value": "true"
            },
            {
              "name": "ID_WEB_SERVICE_URL",
              "value": "http://localhost:6500"
            },
            {
              "name": "ID_API_SERVICE_URL",
              "value": "http://localhost:6501/api"
            },
            {
              "name": "ID_UMS_SERVICE_URL",
              "value": "http://localhost:6502/ums"
            },
            {
              "name": "BI_WEB_SERVICE_URL",
              "value": "http://localhost:6504/bi"
            },
            {
              "name": "BI_API_SERVICE_URL",
              "value": "http://localhost:6505/bi/api"
            },
            {
              "name": "BI_JOBS_SERVICE_URL",
              "value": "http://localhost:6506/bi/jobs"
            },
            {
              "name": "BI_DESIGNER_SERVICE_URL",
              "value": "http://localhost:6507/bi/designer"
            },
            {
              "name": "BI_DESIGNER_HELPER_SERVICE_URL",
              "value": "http://localhost:6507/bi/designer/helper"
            },
            {
              "name": "BOLD_ETL_SERVICE_URL",
              "value": "http://localhost:6509"
            },
            {
              "name": "BOLD_ETL_BLAZOR_SERVICE_URL",
              "value": "http://localhost:6509/framework/blazor.server.js"
            },
            {
              "name": "BOLD_AI_SERVICE_URL",
              "value": "http://localhost:6510/aiservice"
            },
            {
              "name": "REPORTS_WEB_SERVICE_URL",
              "value": "http://localhost:6504/reporting"
            },
            {
              "name": "REPORTS_API_SERVICE_URL",
              "value": "http://localhost:6505/reporting/api"
            },
            {
              "name": "REPORTS_JOBS_SERVICE_URL",
              "value": "http://localhost:6506/reporting/jobs"
            },
            {
              "name": "REPORTS_DESIGNER_SERVICE_URL",
              "value": "http://localhost:6508/reporting/reportservice"
            },
            {
              "name": "REPORTS_VIEWER_SERVICE_URL",
              "value": "http://localhost:6507/reporting/viewer"
            }
          ]
        }
      }
    },
    {
      "type": "Microsoft.Insights/diagnosticSettings",
      "apiVersion": "2021-05-01-preview",
      "name": "appservice-diagnostics",
      "scope": "[resourceId('Microsoft.Web/sites', parameters('appServiceName'))]",
      "dependsOn": [
        "[resourceId('Microsoft.Web/sites', parameters('appServiceName'))]",
        "[resourceId('Microsoft.OperationalInsights/workspaces', variables('workspaceName'))]"
      ],
      "properties": {
        "workspaceId": "[resourceId('Microsoft.OperationalInsights/workspaces', variables('workspaceName'))]",
        "logs": [
          {
            "category": "AppServiceConsoleLogs",
            "enabled": true
          },
          {
            "category": "AppServiceHTTPLogs",
            "enabled": true
          },
          {
            "category": "AppServiceAppLogs",
            "enabled": true
          },
          {
            "category": "AppServicePlatformLogs",
            "enabled": true
          },
          {
            "category": "AppServiceAuditLogs",
            "enabled": true
          }
        ],
        "metrics": [
          {
            "category": "AllMetrics",
            "enabled": true
          }
        ]
      }
    }
  ]
}
```

</details>

---

### Step 4: Fill in Deployment Details

Provide the following parameters:

#### Mandatory (always required)

| Parameter | Description |
|------------|-------------|
| **Subscription** | Choose your Azure subscription. |
| **Resource group** | Select an existing resource group or create a new one. |
| **Region** | Choose the Azure region for deployment. The storage account, app service plan, and app service should all be in the same region. |
| **App Service Name** | Enter a unique name for the Bold BI App URL (3–24 characters, lowercase letters and numbers only). If the name is already taken, the deployment fails — choose another. |
| **App Service Plan Size** | Select the App Service SKU. Available values: <br>• `P1V3_2Core_8GB_DEV` <br>• `P2V3_4Core_16GB_PROD` <br>• `P3V3_8Core_32GB_PROD` |
| **Storage Account Name** | The name of the **existing** storage account that was previously used with the Windows-based App Service. |
| **Storage Account Access Key** | The **access key** of the existing storage account. Find it under the storage account → *Security + networking* → *Access keys*. |
| **Azure Blob Container Name** | The name of the **existing** blob container in the storage account (for example, `bold-services`). |
| **Protocol** | Choose the protocol used to host Bold BI. <br>• Select `http` if you want to host Bold BI over **HTTP** (e.g., `http://<appname>.azurewebsites.net`). <br>• Select `https` if you want to host Bold BI over **HTTPS** (e.g., `https://<appname>.azurewebsites.net`). <br>Make sure the protocol you select here matches how you will access the application later. |
| **Image Name** | The Bold BI Docker image to deploy. <br>• **Default value:** `asme123/boldbi:azure_app_services_images4` <br>• If you have a different Bold BI image, update this value. <br>Example format: `<repository>/<image>:<tag>`. |

  ![mandatory](./images/mandatory_value.png)

---

### Step 5: Review and Create

1. After all details are filled in, click **Review + Create**.
  ![review_and_save](./images/review_create.png)

2. Wait for validation to complete.

3. Click **Create** to begin the deployment.

---

### Step 6: Wait for Resources to Be Created

1. Once the deployment finishes, click **Go to resource group**.
  ![resource_group](./images/resource_group.png)

2. From the resource group, select the **App Service**.
  ![app_service](./images/app_service.png)
---

### Step 7: Wait for the Site to Be Healthy

1. Wait approximately **10 minutes** for the site to come up and run.
2. On the **Overview** page, check the **Health Check** status:
   - `Health Check: 100.00% (Healthy 1 / Degraded 0)` indicates the site is healthy.
   ![health_check](./images/health_check.png)

---

### Step 8: Access the Bold BI Application

1. On the **Overview** page, copy the **Default Domain** URL.

   ![health_check](./images/default_domain.png)

2. Open a browser and navigate to the following URL by appending `/ums/administration/proxy-settings` to the **Default Domain**:

   > **Note:**
   >
   > - If you selected **HTTP**, access:
   >   `http://<Default-Domain>/ums/administration/proxy-settings`
   >
   > - If you selected **HTTPS**, access:
   >   `https://<Default-Domain>/ums/administration/proxy-settings`
   >
   > Wait for the page to load completely before proceeding.

3. If you are redirected to the Bold BI login page, sign in using your Bold BI username and password.

4. After accessing the **Proxy Settings** page, update the **Site URL** or **Proxy URL** field with your new domain URL.

---

