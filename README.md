# Azure Multi-Region Infrastructure Deployment via Bicep

## Overview

This repository serves as a practical implementation and reference guide for concepts learned in the Microsoft Learn Azure Bicep paths. It provisions a multi-region Azure environment featuring SQL databases, Virtual Networks, and a Node.js App Service. The entire deployment is fully automated using GitHub Actions.

## Architecture & Design Decisions

I deliberately avoided hardcoding single resources to practice advanced Bicep capabilities, specifically focusing on **Loops** and **Conditions**.

* **Multi-Region Databases:** Bicep concurrently deploys identical, globally unique SQL Servers and databases to all regions simultaneously without duplicating code.
* **Virtual Networks & Subnets:** The Virtual Networks are deployed alongside the databases in every specified region, `frontend` and `backend` subnets are also generated.
* **Conditional Deployments:** SQL auditing settings and their associated Storage Accounts are conditionally deployed when the deployment environment is set to `prod`.
* **Secure Parameters:** Administrator passwords are never hardcoded. The deployment relies on Azure Key Vault references within the parameter files to securely inject secrets at runtime.
* **Idempotency:** The GitHub Actions workflow dynamically generates the parameter files and uses a consistent Key Vault name based on the repository ID, allowing for continuous, safe re-deployments without creating duplicate resources.

## Deployment Guide (GitHub Actions CI/CD)

The manual `deploy.sh` script has been replaced with a fully automated continuous integration and deployment pipeline located in `.github/workflows/deploy.yml`.

### Prerequisites

1. An active Azure Subscription.
2. Azure CLI installed locally (for initial setup).
3. A GitHub repository to host this code.

### 1. Azure Role-Based Access Control (RBAC) Setup

You need to create a Service Principal for GitHub Actions to authenticate to Azure and give it permissions to manage secrets in Key Vaults.

1. **Get your Subscription ID:**
   ```bash
   az account show --query id -o tsv
   ```
2. **Create the Service Principal:** (Run this in PowerShell or CMD, not Git Bash)
   ```powershell
   az ad sp create-for-rbac --name "GitHubActionsDeploy" --role contributor --scopes /subscriptions/<YOUR_SUBSCRIPTION_ID> --sdk-auth
   ```
   *Copy the entire JSON output provided by this command.*
3. **Grant Key Vault Permissions:** The Contributor role cannot read/write Key Vault secrets. You must explicitly assign the `Key Vault Secrets Officer` role to the newly created Service Principal. 
   Find the `clientId` (or `appId`) from the JSON output and run:
   ```powershell
   az role assignment create --role "Key Vault Secrets Officer" --assignee "<YOUR_SERVICE_PRINCIPAL_CLIENT_ID>" --scope /subscriptions/<YOUR_SUBSCRIPTION_ID>
   ```

### 2. GitHub Secrets Configuration

Navigate to your GitHub repository -> **Settings** -> **Secrets and variables** -> **Actions** -> **New repository secret**.

Add the following two secrets:
* `AZURE_CREDENTIALS`: Paste the exact JSON output from the `az ad sp create-for-rbac` step above.
* `SQL_ADMIN_PASSWORD`: Enter a complex password (e.g., `MySuperSecretPassword123!`). Do **not** wrap it in quotes.

### 3. Run the Deployment

1. Go to the **Actions** tab in your GitHub repository.
2. Select the **Deploy Infrastructure and Application** workflow.
3. Click **Run workflow** (or simply `git push` to the `main` branch).

The workflow will automatically:
1. Create the Resource Group and a globally unique Key Vault.
2. Save your SQL admin password in the Key Vault.
3. Dynamically build the `main.parameters.dev.json` file so you don't have to manage placeholders.
4. Validate and deploy the Azure Bicep templates incrementally.
5. Zip the Node.js application (`src/`) and deploy it to the dynamically generated App Service.

## Developer Notes & Troubleshooting

**Azure App Service 403 Errors:** By default, Bicep provisions a Windows App Service. If your Node.js application encounters a 403 directory browsing error, ensure a valid `package.json` file exists in the `src/` directory. The presence of `package.json` signals Azure (Kudu) to automatically configure `iisnode` and correctly route traffic to your Node.js app instead of treating it as a static website.

**Azure Free Trial Quotas:** I deployed on an Azure Free Trial subscription and encountered `SubscriptionIsOverQuotaForSku` errors for `B1` or `F1` App Service plans. Microsoft strictly limits these compute instances globally on trial accounts. Stepping away and letting the backend rate limits and quota blocks cool off for a few days is how I handled the Azure Free Trial constraints.
