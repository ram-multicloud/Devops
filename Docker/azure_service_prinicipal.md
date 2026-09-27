# Azure Service Principal with Contributor Role

## Overview

An **Azure Service Principal** is an identity used by applications, automation tools, CI/CD pipelines, scripts, and services to authenticate with Azure without using a personal user account.

> **Note (Git Bash / MINGW64 users):** Git Bash auto-converts any string starting with `/` (like `/subscriptions/...`) into a Windows filesystem path, which breaks commands using scopes. Prefix such commands with `MSYS_NO_PATHCONV=1` to disable this behavior. This is already added to the relevant commands below.

In this example, we will:

1. Create an Azure Service Principal.
2. Assign the **Contributor** role.
3. Get the required credentials.
4. Log in to Azure CLI using the Service Principal.
5. Verify the login and permissions.

---

## Prerequisites

Make sure you have:

- An Azure subscription.
- Azure CLI installed.
- Permission to create service principals and assign roles.

Check Azure CLI:

```bash
az version
```

Log in to Azure using your normal Azure account first:

```bash
az login
```

Check your subscriptions:

```bash
az account list --output table
```

Set the subscription you want to use:

```bash
az account set --subscription "<SUBSCRIPTION_ID>"
```

Verify the current subscription:

```bash
az account show --output table
```

---

## Step 1: Create a Service Principal

Run the following command:

```bash
MSYS_NO_PATHCONV=1 az ad sp create-for-rbac \
  --name "my-demo-service-principal" \
  --role "Contributor" \
  --scopes "/subscriptions/<SUBSCRIPTION_ID>"
```

Replace:

```text
<SUBSCRIPTION_ID>
```

with your Azure subscription ID.

### Example

```bash
MSYS_NO_PATHCONV=1 az ad sp create-for-rbac \
  --name "my-demo-service-principal" \
  --role "Contributor" \
  --scopes "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

The command returns JSON similar to:

```json
{
  "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "displayName": "my-demo-service-principal",
  "password": "xxxxxxxxxxxxxxxxxxxxxxxx",
  "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> **Important:** The `password` (client secret) is sensitive. Save it securely. Azure does not display the same secret again after creation.

---

## Step 2: Understand the Service Principal Credentials

The output contains the values required for Azure CLI authentication:

| Value | Meaning |
|---|---|
| `appId` | Client ID / Application ID |
| `password` | Client secret |
| `tenant` | Microsoft Entra tenant ID |
| Subscription ID | Azure subscription where the role is assigned |

You can use these values with:

```bash
az login --service-principal \
  --username "<APP_ID>" \
  --password "<CLIENT_SECRET>" \
  --tenant "<TENANT_ID>"
```

---

## Step 3: Assign the Contributor Role

If you did not assign the role while creating the Service Principal, you can assign it separately.

First get the Service Principal object ID:

```bash
az ad sp list \
  --display-name "my-demo-service-principal" \
  --query "[0].id" \
  --output tsv
```

Store the returned object ID.

Then assign the Contributor role:

```bash
MSYS_NO_PATHCONV=1 az role assignment create \
  --assignee-object-id "<SERVICE_PRINCIPAL_OBJECT_ID>" \
  --assignee-principal-type ServicePrincipal \
  --role "Contributor" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>"
```

### Verify the Role Assignment

```bash
MSYS_NO_PATHCONV=1 az role assignment list \
  --assignee "<APP_ID>" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>" \
  --output table
```

You should see:

```text
Role          Scope
------------  -----------------------------------------------
Contributor   /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

---

## Step 4: Login to Azure CLI Using the Service Principal

Log out of the current Azure CLI session:

```bash
az logout
```

Now log in using the Service Principal:

```bash
az login --service-principal \
  --username "<APP_ID>" \
  --password "<CLIENT_SECRET>" \
  --tenant "<TENANT_ID>"
```

Example:

```bash
az login --service-principal \
  --username "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  --password "YOUR_CLIENT_SECRET" \
  --tenant "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

---

## Step 5: Verify the Azure CLI Login

Check the currently logged-in account:

```bash
az account show --output table
```

You can also check the subscription:

```bash
az account show \
  --query "{Subscription:name, SubscriptionId:id, TenantId:tenantId}" \
  --output table
```

---

## Step 6: Test Contributor Permissions

For example, list resource groups:

```bash
az group list --output table
```

You can also test resource creation if you have a suitable test resource group.

For example:

```bash
az group create \
  --name "sp-demo-rg" \
  --location "eastus"
```

If the Service Principal has Contributor access at the subscription scope, it can manage resources within that subscription.

---

## Role Scope

The Contributor role can be assigned at different scopes.

### Subscription Scope

```bash
--scope "/subscriptions/<SUBSCRIPTION_ID>"
```

This gives Contributor access across the subscription.

### Resource Group Scope

```bash
--scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP_NAME>"
```

This limits access to a specific resource group.

### Resource Scope

You can also assign a role to an individual resource:

```bash
--scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP_NAME>/providers/<RESOURCE_PROVIDER>/<RESOURCE_TYPE>/<RESOURCE_NAME>"
```

Using the smallest required scope is generally preferable for automation.

> On Git Bash, prefix any command using a `--scope` or `--scopes` value with `MSYS_NO_PATHCONV=1` to prevent path mangling.

---

## Complete Example

### 1. Login

```bash
az login
```

### 2. Select Subscription

```bash
az account set --subscription "<SUBSCRIPTION_ID>"
```

### 3. Create Service Principal + Contributor Role

```bash
MSYS_NO_PATHCONV=1 az ad sp create-for-rbac \
  --name "my-demo-service-principal" \
  --role "Contributor" \
  --scopes "/subscriptions/<SUBSCRIPTION_ID>"
```

### 4. Logout

```bash
az logout
```

### 5. Login Using Service Principal

```bash
az login --service-principal \
  --username "<APP_ID>" \
  --password "<CLIENT_SECRET>" \
  --tenant "<TENANT_ID>"
```

### 6. Verify

```bash
az account show --output table
```

### 7. Test

```bash
az group list --output table
```

---

## Environment Variables

For scripts and CI/CD pipelines, you can store the credentials in environment variables instead of putting them directly into commands.

### Linux / macOS

```bash
export AZURE_CLIENT_ID="<APP_ID>"
export AZURE_CLIENT_SECRET="<CLIENT_SECRET>"
export AZURE_TENANT_ID="<TENANT_ID>"
export AZURE_SUBSCRIPTION_ID="<SUBSCRIPTION_ID>"
```

Then:

```bash
az login \
  --service-principal \
  --username "$AZURE_CLIENT_ID" \
  --password "$AZURE_CLIENT_SECRET" \
  --tenant "$AZURE_TENANT_ID"
```

### Windows PowerShell

```powershell
$env:AZURE_CLIENT_ID="<APP_ID>"
$env:AZURE_CLIENT_SECRET="<CLIENT_SECRET>"
$env:AZURE_TENANT_ID="<TENANT_ID>"
$env:AZURE_SUBSCRIPTION_ID="<SUBSCRIPTION_ID>"
```

Then:

```powershell
az login `
  --service-principal `
  --username $env:AZURE_CLIENT_ID `
  --password $env:AZURE_CLIENT_SECRET `
  --tenant $env:AZURE_TENANT_ID
```

> **Note:** PowerShell does not have the Git Bash path-mangling issue, so `MSYS_NO_PATHCONV=1` is not needed there.

---

## Important Security Notes

- Do not commit the Service Principal secret to Git.
- Do not put the client secret directly into source code.
- Do not share the client secret in screenshots or chat.
- Store secrets in a secure secret-management system.
- Use the smallest practical RBAC scope.
- For Azure-hosted workloads, consider using **Managed Identity** instead of a client secret where possible.
- Rotate or remove credentials that are no longer required.

---

## Useful Commands

### List Service Principals

```bash
az ad sp list --all --output table
```

### Find a Service Principal

```bash
az ad sp list \
  --display-name "my-demo-service-principal" \
  --output table
```

### List Role Assignments

```bash
az role assignment list \
  --assignee "<APP_ID>" \
  --output table
```

### Remove the Role Assignment

```bash
MSYS_NO_PATHCONV=1 az role assignment delete \
  --assignee "<APP_ID>" \
  --role "Contributor" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>"
```

### Delete the Service Principal

```bash
az ad sp delete --id "<APP_ID>"
```

---

## Summary

The main workflow is:

```text
Azure Account
     |
     v
az login
     |
     v
Create Service Principal
     |
     v
Assign Contributor Role
     |
     v
Get App ID + Client Secret + Tenant ID
     |
     v
az logout
     |
     v
az login --service-principal
     |
     v
Verify Azure Subscription
```

The key login command is:

```bash
az login --service-principal \
  --username "<APP_ID>" \
  --password "<CLIENT_SECRET>" \
  --tenant "<TENANT_ID>"
```

> **Reminder:** On Git Bash (MINGW64), any command with a `--scope`/`--scopes` value starting with `/subscriptions/...` needs the `MSYS_NO_PATHCONV=1` prefix to avoid the path being rewritten to a Windows filesystem path.
