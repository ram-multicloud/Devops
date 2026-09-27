# Azure Container Apps (ACA) & Azure Container Instances (ACI) — Portal Guide

## What They Are

**Azure Container Instances (ACI)** — runs a single container or container group quickly, with no orchestration, scaling, or ingress management. Best for short-lived tasks, batch jobs, quick tests.

**Azure Container Apps (ACA)** — a managed platform (built on Kubernetes under the hood) for running microservices/APIs with autoscaling, revisions, traffic splitting, and built-in ingress. Best for long-running apps and APIs.

| | ACI | ACA |
|---|---|---|
| Scaling | Manual only | Auto-scale, including scale to zero |
| Ingress | Public IP / VNet only | Built-in HTTPS ingress |
| Revisions | Not supported | Supported (multiple revisions, traffic split) |
| Use case | Batch jobs, one-off tasks | Web apps, APIs, microservices |

---

## Part 1: Create Azure Container Instance (ACI) — Portal Steps

1. Sign in to the **Azure Portal**
2. Search **Container Instances** → click **+ Create**
3. **Basics** tab:
   - **Subscription** — select your subscription
   - **Resource group** — select existing or **Create new**
   - **Container name** — enter a name (e.g., `mycontainer`)
   - **Region** — select a region
   - **Availability zones** — leave default (optional)
   - **SKU** — leave **Standard**
   - **Image source** — choose:
     - **Quickstart images** (test images), or
     - **Azure Container Registry**, or
     - **Docker Hub or other registry**
   - If using ACR: select your **Registry**, **Image**, and **Image tag**
   - **OS type** — Linux or Windows
   - **Size** — set CPU cores and Memory (e.g., 1 vCPU, 1 GiB)
4. **Networking** tab:
   - **Networking type** — **Public** (or Private/VNet)
   - **DNS name label** — enter a unique label (gives you a public FQDN)
   - **Ports** — add the port your container listens on (e.g., 80)
5. **Advanced** tab (optional):
   - Add **Environment variables** if needed
   - Add a **Restart policy** (Always / OnFailure / Never)
6. **Review + create** → **Create**

### View the Running Container
1. Go to the created **Container instance** resource
2. **Overview** tab shows **Status**, **IP address**, **FQDN**
3. Left menu → **Containers** → **Logs** tab to view container logs
4. Left menu → **Containers** → **Connect** tab to open a shell inside the container (if supported)

### Make a Change (Update Image)
ACI does **not support in-place image updates** via the Portal. To change the image:
1. Go to the container instance → **Overview** → click **Delete**
2. Confirm deletion
3. Repeat the **Create** steps above with the new image/tag

---

## Part 2: Create Azure Container Apps (ACA) — Portal Steps

### Step 1 — Create a Container Apps Environment (if not existing)
1. Search **Container Apps** → click **+ Create**
2. **Basics** tab:
   - **Subscription** and **Resource group**
   - **Container app name** — enter a name (e.g., `myapp`)
   - **Region** — select a region
   - **Container Apps Environment** — click **Create new**
     - Enter an **Environment name** (e.g., `myACAEnv`)
     - Leave networking/logging defaults unless you need custom VNet
     - Click **Create**

### Step 2 — Configure the App Container
3. **App settings** / **Container** tab:
   - **Use quickstart image** — uncheck if deploying your own image
   - **Name** — container name
   - **Image source** — choose:
     - **Azure Container Registry**
     - **Docker Hub or other registries**
   - Select **Registry**, **Image**, and **Image tag**
   - **CPU and Memory** — select allocation (e.g., 0.5 vCPU, 1 Gi)

### Step 3 — Configure Ingress
4. **Ingress** tab:
   - **Ingress** — toggle **Enabled**
   - **Ingress traffic** — **Accepting traffic from anywhere** (external) or **Limited to Container Apps Environment** (internal)
   - **Target port** — the port your app listens on (e.g., 80)

### Step 4 — Configure Scaling
5. **Scale** tab (if shown, or under app settings after creation):
   - **Min replicas** — e.g., 1 (or 0 for scale-to-zero)
   - **Max replicas** — e.g., 3
   - Add **Scale rules** (HTTP, CPU, custom) if needed

### Step 5 — Review and Create
6. **Review + create** → **Create**

### Get the App URL
1. Go to the created **Container App** resource
2. **Overview** tab → copy the **Application Url**

### View Logs
1. Left menu → **Log stream** (live logs), or
2. Left menu → **Monitoring** → **Logs** for query-based log analytics

---

## Make Changes to an Existing Container App (Portal)

### Deploy a New Image Version (creates a new revision)
1. Go to the Container App → left menu → **Containers**
2. Click **Edit and deploy**
3. Update the **Image** and/or **Tag**
4. Click **Save** — this automatically creates a **new revision**

### Update Scaling
1. Go to the Container App → left menu → **Scale**
2. Adjust **Min replicas**, **Max replicas**, or scale rules
3. Click **Save**

### Manage Revisions & Traffic Splitting
1. Left menu → **Revisions**
2. View all active revisions
3. Click **Manage traffic split**
4. Adjust the **%** allocated to each revision (e.g., 50/50 for canary testing)
5. Click **Save**

### Roll Back to a Previous Revision
1. Left menu → **Revisions**
2. Select the older revision → set its traffic weight to 100%
3. Click **Save**

### Delete a Container App
1. Go to the Container App → **Overview**
2. Click **Delete**
3. Confirm

---

## Quick Reference — Where to Make Changes

| Change | ACI (Portal) | ACA (Portal) |
|---|---|---|
| Update image | Delete + recreate | **Containers** → Edit and deploy |
| Scale replicas | Not supported | **Scale** blade |
| Rollback | Recreate with old image | **Revisions** → shift traffic |
| View logs | **Containers** → Logs tab | **Log stream** / **Monitoring → Logs** |
| Networking/ports | Delete + recreate | **Ingress** settings |
