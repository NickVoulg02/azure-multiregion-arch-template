# Azure Multi-Region Infrastructure Deployment via Bicep

## Overview

This repository serves as a practical implementation and reference guide for concepts learned in the Microsoft Learn Azure Bicep paths. It provisions a multi-region Azure environment featuring SQL databases, Virtual Networks, and an optional Node.js App Service.

## Architecture & Design Decisions

I deliberately avoided hardcoding single resources to practice advanced Bicep capabilities, specifically focusing on **Loops** and **Conditions**.

* **Multi-Region Databases:** Bicep concurrently deploys identical, globally unique SQL Servers and databases to all regions simultaneously without duplicating code.
* **Virtual Networks & Subnets:** The Virtual Networks are deployed alongside the databases in every specified region, `frontend` and `backend` subnets are also generated.
* **Conditional Deployments:** SQL auditing settings and their associated Storage Accounts are conditionally deployed when the deployment environment is set to `prod`.
* **Secure Parameters:** Administrator passwords are never hardcoded. The deployment relies on Azure Key Vault references within the parameter files to securely inject secrets at runtime.

## Deployment Guide

Deploying this project requires provisioning the infrastructure via Bicep and pushing the application code to the created App Service. This process has been consolidated into a single deployment script.

### Prerequisites

1. An active Azure Subscription.
2. Azure CLI installed and authenticated (`az login`).

### Setup & Execution

1. **Prepare Parameters:** Copy `infra/main.parameters.example.json` to a new file named `main.parameters.dev.json`. Update the Key Vault reference to point to your specific Azure Subscription and Key Vault ID.
2. **Configure the Script:** Open `deploy.sh` and update the configuration variables at the top of the file with your specific values:
   ```bash
   KEY_VAULT_NAME="<YOUR_KEY_VAULT_NAME>"
   SQL_ADMIN_PASSWORD="<YOUR_COMPLEX_PASSWORD>"
   APP_SERVICE_NAME="<DEPLOYED_APP_SERVICE_NAME>"
   ```

3. **Make the Script Executable:**
   ```bash
   chmod +x deploy.sh
   ```

4. **Run the Deployment:**
    ```bash
    ./deploy.sh
    ```

*Note: The script will pause during execution to verify that you have assigned the required Key Vault role and updated the parameter file.*

## Developer Notes & Troubleshooting

**Azure Free Trial Quotas:** I deployed on an Azure Free Trial subscription and encountered `SubscriptionIsOverQuotaForSku` errors for `B1` or `F1` App Service plans. Microsoft strictly limits these compute instances globally on trial accounts.

Stepping away and letting the backend rate limits and quota blocks cool off for a few days is how I handled the Azure Free Trial constraints.
