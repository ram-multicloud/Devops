# Azure Container Apps (ACA) vs Azure Container Instances (ACI)

## Overview

| | **Azure Container Apps (ACA)** | **Azure Container Instances (ACI)** |
|---|---|---|
| Purpose | Run microservices/apps with scaling, revisions, ingress | Run single/simple containers quickly, no orchestration |
| Scaling | Auto-scale (incl. scale to zero), KEDA-based | Manual — no built-in autoscaling |
| Networking | Built-in ingress, HTTPS, custom domains | Public IP or VNet, no built-in ingress/load balancing |
| Revisions | Yes — supports multiple revisions, traffic splitting | No revisions — replace/redeploy the container |
| Orchestration | Managed Kubernetes-based environment (hidden from user) | No orchestration — just runs the container |
| Best for | Web apps, APIs, event-driven microservices, long-running services | Batch jobs, short-lived tasks, CI/CD build agents, quick tests |
| Pricing model | Pay per vCPU/memory usage + scale | Pay per second per container group |

**Rule of thumb:** Use **ACI** for quick, isolated, short-lived containers. Use **ACA** for apps that need scaling, revisions, and ingress management.

---

## Part 1: Azure Container Instances (ACI)

### Step 1 — Create a Resource Group (if not existing)
```bash
az group create --name myRG --location eastus
```

### Step 2 — Create a Container Instance
```bash
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image <registryName>.azurecr.io/myimage:tag \
  --cpu 1 \
  --memory 1 \
  --registry-login-server <registryName>.azurecr.io \
  --registry-username <acr-username> \
  --registry-password <acr-password> \
  --dns-name-label mycontainerapp \
  --ports 80
```

### Step 3 — Check Status
```bash
az container show \
  --resource-group myRG \
  --name mycontainer \
  --query "{FQDN:ipAddress.fqdn, State:instanceView.state}" \
  --output table
```

### Step 4 — View Logs
```bash
az container logs --resource-group myRG --name mycontainer
```

### Step 5 — Make a Change (update image/config)
ACI has **no in-place update** for the image — you delete and recreate:
```bash
az container delete --resource-group myRG --name mycontainer --yes

az container create \
  --resource-group myRG \
  --name mycontainer \
  --image <registryName>.azurecr.io/myimage:newtag \
  --cpu 1 \
  --memory 1 \
  --registry-login-server <registryName>.azurecr.io \
  --registry-username <acr-username> \
  --registry-password <acr-password> \
  --dns-name-label mycontainerapp \
  --ports 80
```

### Step 6 — Delete the Container Instance
```bash
az container delete --resource-group myRG --name mycontainer --yes
```

---

## Part 2: Azure Container Apps (ACA)

### Step 1 — Install/Update the Container Apps CLI Extension
```bash
az extension add --name containerapp --upgrade
```

### Step 2 — Register Required Providers (one-time per subscription)
```bash
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

### Step 3 — Create a Resource Group
```bash
az group create --name myRG --location eastus
```

### Step 4 — Create a Container Apps Environment
```bash
az containerapp env create \
  --name myACAEnv \
  --resource-group myRG \
  --location eastus
```

### Step 5 — Create the Container App
```bash
az containerapp create \
  --name myapp \
  --resource-group myRG \
  --environment myACAEnv \
  --image <registryName>.azurecr.io/myimage:tag \
  --registry-server <registryName>.azurecr.io \
  --registry-username <acr-username> \
  --registry-password <acr-password> \
  --target-port 80 \
  --ingress external \
  --cpu 0.5 --memory 1.0Gi \
  --min-replicas 1 --max-replicas 3
```

### Step 6 — Get the App URL
```bash
az containerapp show \
  --name myapp \
  --resource-group myRG \
  --query properties.configuration.ingress.fqdn \
  --output tsv
```

### Step 7 — Make a Change (deploy a new image → creates a new revision)
```bash
az containerapp update \
  --name myapp \
  --resource-group myRG \
  --image <registryName>.azurecr.io/myimage:newtag
```
This creates a **new revision** automatically — old revision stays available until traffic is fully shifted (unless `--revision-mode Single`, which is the default and replaces traffic to the new revision immediately).

### Step 8 — Update Scaling Rules
```bash
az containerapp update \
  --name myapp \
  --resource-group myRG \
  --min-replicas 2 \
  --max-replicas 5
```

### Step 9 — List Revisions
```bash
az containerapp revision list \
  --name myapp \
  --resource-group myRG \
  --output table
```

### Step 10 — Split Traffic Between Revisions (multiple revision mode)
```bash
az containerapp ingress traffic set \
  --name myapp \
  --resource-group myRG \
  --revision-weight <revision-1>=50 <revision-2>=50
```

### Step 11 — View Logs
```bash
az containerapp logs show \
  --name myapp \
  --resource-group myRG \
  --follow
```

### Step 12 — Delete the Container App
```bash
az containerapp delete --name myapp --resource-group myRG --yes
```

---

## Quick Reference — Making Changes

| Change | ACI | ACA |
|---|---|---|
| Update image | Delete + recreate | `az containerapp update --image ...` (new revision) |
| Scale | Not supported (fixed) | `az containerapp update --min-replicas --max-replicas` |
| Rollback | Recreate with old image | Shift traffic back to previous revision |
| View logs | `az container logs` | `az containerapp logs show` |
| Networking change | Delete + recreate | `az containerapp ingress update` |
