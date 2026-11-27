# Ansible Dynamic Inventory for Azure

A practical guide covering how dynamic inventory works, the prerequisites (Service Principal, roles, permissions), and a worked example that targets VMs tagged `role=webserver`.

---

## Table of Contents

1. [What is a dynamic inventory?](#1-what-is-a-dynamic-inventory)
2. [How the Azure inventory plugin works](#2-how-the-azure-inventory-plugin-works)
3. [Prerequisites](#3-prerequisites)
4. [Service Principal (SP)](#4-service-principal-sp)
5. [Roles and permissions](#5-roles-and-permissions)
6. [Tagging VMs (the `webserver` example)](#6-tagging-vms-the-webserver-example)
7. [Creating the inventory file](#7-creating-the-inventory-file)
8. [Testing the inventory](#8-testing-the-inventory)
9. [Using it in a playbook](#9-using-it-in-a-playbook)
10. [Other authentication methods](#10-other-authentication-methods)
11. [Performance and caching](#11-performance-and-caching)
12. [Security best practices](#12-security-best-practices)
13. [Troubleshooting](#13-troubleshooting)
14. [Quick reference](#14-quick-reference)

---

## 1. What is a dynamic inventory?

### Static inventory

A static inventory is a file you maintain by hand:

```ini
[webservers]
10.0.1.4
10.0.1.5

[dbservers]
10.0.2.4
```

This breaks down in the cloud because VMs are created, deleted, resized, and re-addressed constantly. Keeping the file in sync is manual and error-prone.

### Dynamic inventory

A dynamic inventory asks the cloud provider "what exists right now?" every time Ansible runs, and builds the host list and groups automatically.

| Feature | Static inventory | Dynamic inventory |
|---|---|---|
| Source of truth | A file you edit | Azure itself |
| New VM appears | You add it manually | Picked up automatically |
| Deleted VM | Stale entry remains | Disappears automatically |
| Grouping | Manual | Based on tags, location, resource group, etc. |
| Best for | Small, fixed environments | Cloud and autoscaling environments |

---

## 2. How the Azure inventory plugin works

Ansible uses the **`azure.azcollection.azure_rm`** inventory plugin, part of the `azure.azcollection` collection.

```
 ┌──────────────┐    1. read inventory file     ┌───────────────────────┐
 │ ansible-     │ ────────────────────────────► │ inventory.azure_rm.yml│
 │ playbook     │                               └───────────────────────┘
 │              │    2. authenticate (SP / CLI / MSI)
 │              │ ────────────────────────────► ┌───────────────────────┐
 │              │    3. list VMs, NICs, IPs     │  Azure Resource       │
 │              │ ◄──────────────────────────── │  Manager (ARM) API    │
 │              │                               └───────────────────────┘
 │              │    4. build hosts + groups in memory
 │              │    5. run tasks against the hosts
 └──────────────┘
```

Steps in detail:

1. Ansible reads your `*.azure_rm.yml` file.
2. The plugin authenticates to Azure.
3. It calls Azure Resource Manager to list VMs and related network resources (NICs, public IPs).
4. It converts them into hosts, host variables, and groups (using `keyed_groups`, `conditional_groups`, and so on).
5. Your playbook runs against the resulting inventory.

> The plugin is **read-only**. It only needs permission to *read* resources.

---

## 3. Prerequisites

| Requirement | Details |
|---|---|
| Control node | Linux or macOS (or WSL) with Ansible installed |
| Python | 3.9+ recommended |
| Ansible | `ansible-core` 2.14+ recommended |
| Azure subscription | At least one subscription with VMs |
| Azure identity | A Service Principal, user login via Azure CLI, or a Managed Identity |
| Network access | Control node can reach Azure APIs (`management.azure.com`) and the VM IPs you want to manage |

### Install Ansible and the Azure collection

```bash
# Optional: use a virtualenv
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate

pip install ansible

# Azure collection
ansible-galaxy collection install azure.azcollection

# Python SDK dependencies required by the collection
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
```

Verify:

```bash
ansible-galaxy collection list | grep azure
ansible --version
```

---

## 4. Service Principal (SP)

### What is it?

A **Service Principal** is a non-human identity in Microsoft Entra ID (formerly Azure AD) that applications and automation tools use to sign in to Azure. Think of it as a "service account" for Ansible.

Why use one instead of your own login?

- It is not tied to a person who may leave the company.
- You can give it only the permissions it needs.
- It works non-interactively (CI/CD, cron, AWX/Tower).
- It can be audited and its secret rotated independently.

### The four values you need

| Value | Environment variable | Where it comes from |
|---|---|---|
| Subscription ID | `AZURE_SUBSCRIPTION_ID` | Azure Portal or `az account show` |
| Client ID (App ID) | `AZURE_CLIENT_ID` | SP creation output (`appId`) |
| Client Secret | `AZURE_SECRET` | SP creation output (`password`) |
| Tenant ID | `AZURE_TENANT` | SP creation output (`tenant`) |

### Create the SP (Azure CLI)

```bash
# Log in and select the subscription
az login
az account set --subscription "<SUBSCRIPTION_ID>"

SUB_ID=$(az account show --query id -o tsv)

# Create the SP with Reader access on the whole subscription
az ad sp create-for-rbac \
  --name "sp-ansible-inventory" \
  --role "Reader" \
  --scopes "/subscriptions/$SUB_ID"
```

Sample output:

```json
{
  "appId": "11111111-2222-3333-4444-555555555555",
  "displayName": "sp-ansible-inventory",
  "password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "tenant": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
}
```

> **Save the `password` immediately.** It is shown only once. If lost, reset it with `az ad sp credential reset`.

Mapping the output:

| CLI output | Ansible / environment variable |
|---|---|
| `appId` | `AZURE_CLIENT_ID` |
| `password` | `AZURE_SECRET` |
| `tenant` | `AZURE_TENANT` |
| (your subscription) | `AZURE_SUBSCRIPTION_ID` |

### Create the SP (Azure Portal alternative)

1. **Microsoft Entra ID** → **App registrations** → **New registration**.
2. Name it `sp-ansible-inventory` and register.
3. Copy **Application (client) ID** and **Directory (tenant) ID**.
4. **Certificates & secrets** → **New client secret** → copy the **Value** (not the Secret ID).
5. Assign a role (see next section).

### Export credentials

```bash
export AZURE_SUBSCRIPTION_ID="<subscription-id>"
export AZURE_CLIENT_ID="<appId>"
export AZURE_SECRET="<password>"
export AZURE_TENANT="<tenant>"
```

Or use a credentials file at `~/.azure/credentials`:

```ini
[default]
subscription_id=<subscription-id>
client_id=<appId>
secret=<password>
tenant=<tenant>
```

```bash
chmod 600 ~/.azure/credentials
```

---

## 5. Roles and permissions

Azure uses **RBAC (role-based access control)**. A role assignment has three parts:

```
WHO (security principal)  +  WHAT (role definition)  +  WHERE (scope)
   sp-ansible-inventory       Reader                    /subscriptions/<id>
```

### Scope levels (broad to narrow)

| Scope | Example path |
|---|---|
| Management group | `/providers/Microsoft.Management/managementGroups/<mg>` |
| Subscription | `/subscriptions/<sub-id>` |
| Resource group | `/subscriptions/<sub-id>/resourceGroups/rg-web-prod` |
| Resource | `/subscriptions/<sub-id>/resourceGroups/rg-web-prod/providers/Microsoft.Compute/virtualMachines/vm-web-01` |

Use the **narrowest scope** that works. If you only inventory two resource groups, assign the role on those two resource groups, not the whole subscription.

### Which role do I need?

| Use case | Role | Notes |
|---|---|---|
| Dynamic inventory only | **Reader** | Simplest. Read-only on everything in scope. |
| Inventory only, strict least privilege | **Custom role** (below) | Only the read actions the plugin needs. |
| Inventory plus managing VMs (start, stop, create) | **Virtual Machine Contributor** plus **Network Contributor** as needed | Only if playbooks also use Azure modules. |
| Full resource management | **Contributor** | Avoid unless necessary. |

> Running playbooks over SSH or WinRM against the VMs does **not** require Azure permissions. The SP is only used to *discover* hosts. SSH access is separate (keys or passwords on the VMs).

### Assign Reader at subscription scope

```bash
az role assignment create \
  --assignee "<appId>" \
  --role "Reader" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>"
```

### Assign Reader at resource group scope (recommended)

```bash
az role assignment create \
  --assignee "<appId>" \
  --role "Reader" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/rg-web-prod"
```

### Verify the assignment

```bash
az role assignment list \
  --assignee "<appId>" \
  --all \
  --output table
```

### Custom least-privilege role (optional)

The inventory plugin needs to read VMs, network interfaces, public IPs, and resource groups. Save as `ansible-inventory-reader.json`:

```json
{
  "Name": "Ansible Inventory Reader",
  "IsCustom": true,
  "Description": "Read-only access needed by the Ansible azure_rm inventory plugin.",
  "Actions": [
    "Microsoft.Resources/subscriptions/read",
    "Microsoft.Resources/subscriptions/resourceGroups/read",
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachineScaleSets/read",
    "Microsoft.Compute/virtualMachineScaleSets/virtualMachines/read",
    "Microsoft.Network/networkInterfaces/read",
    "Microsoft.Network/publicIPAddresses/read",
    "Microsoft.Network/virtualNetworks/read"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/<SUBSCRIPTION_ID>"
  ]
}
```

Create and assign it:

```bash
az role definition create --role-definition @ansible-inventory-reader.json

az role assignment create \
  --assignee "<appId>" \
  --role "Ansible Inventory Reader" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/rg-web-prod"
```

> If the inventory returns empty or partial results with a custom role, add the missing `*/read` action. The built-in **Reader** role is the safest starting point.

### Permissions summary

| Azure resource type | Why it is needed |
|---|---|
| `Microsoft.Compute/virtualMachines/read` | List VMs, tags, power state, OS, size |
| `Microsoft.Network/networkInterfaces/read` | Find the NIC attached to each VM |
| `Microsoft.Network/publicIPAddresses/read` | Resolve public IPs |
| `Microsoft.Network/virtualNetworks/read` | Resolve network information |
| `Microsoft.Resources/subscriptions/resourceGroups/read` | List resource groups |
| `Microsoft.Compute/virtualMachineScaleSets/*/read` | Include scale set instances |

---

## 6. Tagging VMs (the `webserver` example)

Tags are the most flexible way to group hosts. In this example we tag web VMs with `role=webserver` and `env=prod`.

### Tag at VM creation

```bash
az vm create \
  --resource-group rg-web-prod \
  --name vm-web-01 \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --tags role=webserver env=prod app=nginx
```

### Tag an existing VM

```bash
# Merge tags without overwriting existing ones
az tag update \
  --resource-id "/subscriptions/<SUB_ID>/resourceGroups/rg-web-prod/providers/Microsoft.Compute/virtualMachines/vm-web-01" \
  --operation Merge \
  --tags role=webserver env=prod app=nginx
```

### Suggested tagging scheme

| Tag key | Example values | Purpose |
|---|---|---|
| `role` | `webserver`, `dbserver`, `appserver` | What the VM does |
| `env` | `dev`, `test`, `prod` | Environment |
| `app` | `nginx`, `tomcat`, `postgres` | Application |
| `owner` | `platform-team` | Accountability |

### How tags become Ansible groups

With this `keyed_groups` setting:

```yaml
keyed_groups:
  - prefix: tag
    key: tags
```

The plugin generates groups in the form `tag_<key>_<value>`:

| VM tags | Resulting groups |
|---|---|
| `role=webserver`, `env=prod` | `tag_role_webserver`, `tag_env_prod` |
| `role=dbserver`, `env=prod` | `tag_role_dbserver`, `tag_env_prod` |

So `vm-web-01` lands in both `tag_role_webserver` and `tag_env_prod`. To target production web servers only, use the intersection: `tag_role_webserver:&tag_env_prod`.

---

## 7. Creating the inventory file

The filename **must end with `azure_rm.yml` or `azure_rm.yaml`**, or Ansible will not recognise it. Example: `inventory.azure_rm.yml`.

### Minimal example

```yaml
plugin: azure.azcollection.azure_rm
auth_source: env
```

### Full example for the web server scenario

```yaml
plugin: azure.azcollection.azure_rm

# Authentication: env = read AZURE_* environment variables
# Other options: auto, cli, credential_file, msi
auth_source: env

# Limit the scan to the resource groups you care about
include_vm_resource_groups:
  - rg-web-prod
  - rg-web-dev

# Skip VMs that are stopped or deallocated
exclude_host_filters:
  - powerstate != 'running'

# Use the VM name (not name + random suffix) as the inventory hostname
plain_host_names: true

# Tell Ansible which address to connect to
hostvar_expressions:
  # Private IP, when the control node is inside the VNet or has VPN access
  ansible_host: private_ipv4_addresses[0]
  # For public IPs use instead:
  # ansible_host: public_ipv4_addresses[0]

# Build groups automatically
keyed_groups:
  # tag_role_webserver, tag_env_prod, ...
  - prefix: tag
    key: tags
  # loc_eastus, loc_westeurope, ...
  - prefix: loc
    key: location
  # rg_rg-web-prod, ...
  - prefix: rg
    key: resource_group | lower
  # os_Linux, os_Windows
  - prefix: os
    key: os_disk.operating_system_type

# Build groups from conditions
conditional_groups:
  # Any VM whose name contains "web" goes into "webservers"
  webservers: "'web' in name"
  # Any VM tagged role=webserver also goes into "webservers"
  # (Jinja is evaluated per host; tags is a dictionary)
  tagged_web: "tags.role | default('') == 'webserver'"
```

### Key options explained

| Option | Meaning |
|---|---|
| `plugin` | Must be `azure.azcollection.azure_rm` |
| `auth_source` | How to authenticate (`auto`, `cli`, `env`, `credential_file`, `msi`) |
| `include_vm_resource_groups` | Only inventory these resource groups (default: all, `*`) |
| `exclude_host_filters` | Drop hosts matching the condition |
| `include_host_filters` | Keep only hosts matching the condition |
| `plain_host_names` | Use the plain VM name as the host name |
| `hostvar_expressions` | Create or override host variables using Jinja expressions |
| `keyed_groups` | Auto-create groups from a host variable |
| `conditional_groups` | Create a group when a Jinja condition is true |
| `default_host_filters` | Defaults filter out VMs that are not running or are in a failed state; set to `[]` to disable |

---

## 8. Testing the inventory

### Show the group tree

```bash
ansible-inventory -i inventory.azure_rm.yml --graph
```

Example output:

```
@all:
  |--@tag_role_webserver:
  |  |--vm-web-01
  |  |--vm-web-02
  |--@tag_env_prod:
  |  |--vm-web-01
  |  |--vm-web-02
  |--@webservers:
  |  |--vm-web-01
  |  |--vm-web-02
  |--@os_Linux:
  |  |--vm-web-01
  |  |--vm-web-02
  |--@ungrouped:
```

### Dump everything as JSON

```bash
ansible-inventory -i inventory.azure_rm.yml --list
```

### Inspect variables for a single host

```bash
ansible-inventory -i inventory.azure_rm.yml --host vm-web-01
```

Useful fields in the output: `tags`, `powerstate`, `private_ipv4_addresses`, `public_ipv4_addresses`, `location`, `os_disk`, `image`, `resource_group`.

### Ping the web servers

```bash
ansible -i inventory.azure_rm.yml tag_role_webserver -m ping -u azureuser --private-key ~/.ssh/id_rsa
```

---

## 9. Using it in a playbook

### Install and configure nginx on all web servers

`site.yml`:

```yaml
---
- name: Configure web servers
  hosts: tag_role_webserver
  become: true

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy index page
      ansible.builtin.copy:
        content: "Hello from {{ inventory_hostname }} ({{ location }})\n"
        dest: /var/www/html/index.html
        mode: "0644"
```

Run it:

```bash
ansible-playbook -i inventory.azure_rm.yml site.yml
```

### Target patterns

```bash
# All web servers
ansible-playbook -i inventory.azure_rm.yml site.yml --limit tag_role_webserver

# Only production web servers (intersection)
ansible-playbook -i inventory.azure_rm.yml site.yml --limit 'tag_role_webserver:&tag_env_prod'

# Web servers except dev
ansible-playbook -i inventory.azure_rm.yml site.yml --limit 'tag_role_webserver:!tag_env_dev'
```

### Make it the default inventory

`ansible.cfg`:

```ini
[defaults]
inventory = ./inventory.azure_rm.yml
host_key_checking = False
remote_user = azureuser
private_key_file = ~/.ssh/id_rsa

[inventory]
enable_plugins = azure.azcollection.azure_rm, yaml, ini
```

Then simply run:

```bash
ansible-playbook site.yml
```

---

## 10. Other authentication methods

### Azure CLI (best for local testing)

```bash
az login
```

```yaml
plugin: azure.azcollection.azure_rm
auth_source: cli
```

### Credentials file

```yaml
plugin: azure.azcollection.azure_rm
auth_source: credential_file
# Optional: pick a profile in ~/.azure/credentials
# profile: default
```

### Inline SP values (not recommended in plain text)

```yaml
plugin: azure.azcollection.azure_rm
auth_source: auto
subscription_id: "{{ lookup('env', 'AZURE_SUBSCRIPTION_ID') }}"
client_id: "{{ lookup('env', 'AZURE_CLIENT_ID') }}"
secret: "{{ lookup('env', 'AZURE_SECRET') }}"
tenant: "{{ lookup('env', 'AZURE_TENANT') }}"
```

### Managed Identity (when Ansible runs on an Azure VM)

No secrets to manage. Enable a system-assigned identity on the control VM and give it the Reader role:

```bash
az vm identity assign --name ansible-controller --resource-group rg-tools

PRINCIPAL_ID=$(az vm show --name ansible-controller --resource-group rg-tools \
  --query identity.principalId -o tsv)

az role assignment create \
  --assignee "$PRINCIPAL_ID" \
  --role "Reader" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>"
```

```yaml
plugin: azure.azcollection.azure_rm
auth_source: msi
```

---

## 11. Performance and caching

For large estates every run queries Azure, which can be slow. Enable caching:

```yaml
plugin: azure.azcollection.azure_rm
auth_source: env

cache: true
cache_plugin: ansible.builtin.jsonfile
cache_connection: /tmp/azure_inventory_cache
cache_timeout: 300      # seconds
```

Force a refresh:

```bash
ansible-inventory -i inventory.azure_rm.yml --list --flush-cache
```

Other speed-ups:

- Limit `include_vm_resource_groups` to what you need.
- Use `include_host_filters` and `exclude_host_filters` to reduce the set.
- Use one inventory file per subscription and run them in parallel if needed.

---

## 12. Security best practices

- **Least privilege:** use Reader (or the custom role) and the narrowest scope.
- **Never commit secrets** to Git. Keep `AZURE_SECRET` in environment variables, a secrets manager (Azure Key Vault, HashiCorp Vault), or `ansible-vault`.
- **Set secret expiry and rotate it** regularly:
  ```bash
  az ad sp credential reset --id "<appId>" --years 1
  ```
- **Prefer Managed Identity** when the control node runs in Azure. No secret exists to leak.
- **Prefer certificate credentials** over client secrets when possible.
- **Restrict file permissions:** `chmod 600 ~/.azure/credentials`.
- **Separate identities** per environment (one SP for dev, another for prod).
- **Audit** role assignments periodically:
  ```bash
  az role assignment list --assignee "<appId>" --all -o table
  ```
- **Use separate identities** for discovery (Reader) and for changes (Contributor), so the inventory identity cannot modify anything.

Encrypting a secrets file with Ansible Vault:

```bash
ansible-vault create group_vars/all/azure_vault.yml
```

---

## 13. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Inventory file is not recognised` / plugin not used | Filename does not end with `azure_rm.yml` | Rename the file |
| `No inventory plugins available` or plugin not found | Collection not installed or not enabled | `ansible-galaxy collection install azure.azcollection`; add to `enable_plugins` |
| `ModuleNotFoundError: azure...` | Python SDK dependencies missing | Install `requirements.txt` from the collection into the same Python environment |
| `AuthorizationFailed` | SP has no role or the wrong scope | Check `az role assignment list --assignee <appId> --all` |
| `AADSTS7000215: Invalid client secret` | Wrong or expired secret (using the Secret ID instead of the Value is common) | Reset the credential and use the secret **Value** |
| Inventory is empty | Resource group name typo, VMs are stopped, or filters exclude them | Check `include_vm_resource_groups`; set `default_host_filters: []` to see all VMs |
| Hosts listed but SSH times out | No network path to private IPs | Use VPN, Bastion tunnel, a jump host, or run Ansible inside the VNet |
| `ansible_host` is empty | VM has no public IP and you used `public_ipv4_addresses[0]` | Use `private_ipv4_addresses[0]` |
| Group `tag_role_webserver` missing | Tag key or value differs (case, spelling) | Check with `ansible-inventory --host <vm>` and fix the tag |
| Stale results | Cache still valid | `--flush-cache` or lower `cache_timeout` |

Debug with verbose output:

```bash
ansible-inventory -i inventory.azure_rm.yml --graph -vvv
```

---

## 14. Quick reference

### End-to-end setup in 8 commands

```bash
# 1. Install
pip install ansible
ansible-galaxy collection install azure.azcollection
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt

# 2. Create SP with Reader on the web resource group
az login
SUB_ID=$(az account show --query id -o tsv)
az ad sp create-for-rbac --name sp-ansible-inventory --role Reader \
  --scopes /subscriptions/$SUB_ID/resourceGroups/rg-web-prod

# 3. Export the credentials from the output
export AZURE_SUBSCRIPTION_ID=$SUB_ID
export AZURE_CLIENT_ID=<appId>
export AZURE_SECRET=<password>
export AZURE_TENANT=<tenant>

# 4. Tag the VM
az tag update --resource-id <vm-resource-id> --operation Merge --tags role=webserver env=prod

# 5. Create inventory.azure_rm.yml (see section 7), then test
ansible-inventory -i inventory.azure_rm.yml --graph

# 6. Run
ansible -i inventory.azure_rm.yml tag_role_webserver -m ping -u azureuser
```

### Cheat sheet

| Task | Command or setting |
|---|---|
| Show groups | `ansible-inventory -i inv.azure_rm.yml --graph` |
| Show host vars | `ansible-inventory -i inv.azure_rm.yml --host <name>` |
| Web servers group | `tag_role_webserver` |
| Prod web servers | `tag_role_webserver:&tag_env_prod` |
| Use private IP | `hostvar_expressions: {ansible_host: private_ipv4_addresses[0]}` |
| Only running VMs | `exclude_host_filters: [powerstate != 'running']` |
| Refresh cache | `--flush-cache` |
| Check SP roles | `az role assignment list --assignee <appId> --all -o table` |

### Naming used in this guide

| Item | Example value |
|---|---|
| Service Principal | `sp-ansible-inventory` |
| Role | `Reader` (or custom `Ansible Inventory Reader`) |
| Resource group | `rg-web-prod`, `rg-web-dev` |
| VM names | `vm-web-01`, `vm-web-02` |
| Tag | `role=webserver`, `env=prod` |
| Ansible group | `tag_role_webserver` |
| Inventory file | `inventory.azure_rm.yml` |

### References

- Collection docs: https://docs.ansible.com/ansible/latest/collections/azure/azcollection/azure_rm_inventory.html
- Azure RBAC built-in roles: https://learn.microsoft.com/azure/role-based-access-control/built-in-roles
- Azure CLI `az ad sp`: https://learn.microsoft.com/cli/azure/ad/sp
