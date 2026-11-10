# Ansible Inventory
### What it is, types, examples, and use cases

---

## 1. What is an Ansible Inventory?

An **inventory** is the list of machines (hosts) that Ansible manages, along with how they are organized and how to connect to them. When you run a playbook or an ad-hoc command, Ansible reads the inventory to decide **which hosts to run on**.

An inventory can define:

- **Hosts**: individual servers, VMs, network devices, containers, or cloud instances
- **Groups**: collections of hosts (e.g. `webservers`, `dbservers`)
- **Variables**: settings attached to a host or group (ports, users, passwords, app versions)
- **Nested groups**: groups of groups (e.g. `production` contains `webservers` and `dbservers`)

```
Control node (Ansible)
        │  reads inventory
        ▼
 ┌──────────────┬──────────────┬──────────────┐
 │ webservers   │ dbservers    │ loadbalancers│
 │ web1, web2   │ db1, db2     │ lb1          │
 └──────────────┴──────────────┴──────────────┘
```

**Default location:** `/etc/ansible/hosts`
**Custom location:** use `-i` on the command line or set `inventory` in `ansible.cfg`.

```bash
ansible all -i inventory.ini -m ping
ansible-playbook -i inventory.ini site.yml
```

---

## 2. Types of Inventory

| Type | Description | Best for |
|---|---|---|
| **Static inventory** | A fixed file (INI or YAML) that you edit by hand | Small or stable environments, on-prem servers |
| **Dynamic inventory** | Generated at runtime from a script or plugin that queries a source (cloud API, CMDB) | Cloud and autoscaling environments |
| **Inventory directory** | A directory containing multiple inventory files/scripts combined together | Mixed static and dynamic sources, multiple environments |
| **Inline / ad-hoc inventory** | Hosts passed directly on the command line | Quick tests |

---

## 3. Static Inventory

A static inventory is a plain text file. Two formats are supported: **INI** and **YAML**.

### 3.1 INI format

```ini
# inventory.ini

# Ungrouped host
mail.example.com

[webservers]
web1.example.com
web2.example.com ansible_port=2222

[dbservers]
db1.example.com ansible_user=dbadmin
db2.example.com

[loadbalancers]
lb1.example.com

# Group of groups
[production:children]
webservers
dbservers
loadbalancers

# Variables for a whole group
[webservers:vars]
http_port=80
app_version=2.4.1

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### 3.2 YAML format

```yaml
# inventory.yml
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
  hosts:
    mail.example.com:
  children:
    production:
      children:
        webservers:
          hosts:
            web1.example.com:
            web2.example.com:
              ansible_port: 2222
          vars:
            http_port: 80
            app_version: 2.4.1
        dbservers:
          hosts:
            db1.example.com:
              ansible_user: dbadmin
            db2.example.com:
        loadbalancers:
          hosts:
            lb1.example.com:
```

### 3.3 Host ranges and patterns

```ini
[webservers]
web[01:05].example.com        # web01 to web05

[dbservers]
db-[a:c].example.com          # db-a, db-b, db-c
```

### 3.4 Built-in groups

| Group | Meaning |
|---|---|
| `all` | Every host in the inventory |
| `ungrouped` | Hosts not in any group other than `all` |

---

## 4. Common Inventory (Connection) Variables

| Variable | Purpose | Example |
|---|---|---|
| `ansible_host` | Real IP/DNS to connect to (alias differs from address) | `ansible_host=10.0.1.15` |
| `ansible_port` | SSH port | `ansible_port=2222` |
| `ansible_user` | Remote login user | `ansible_user=ubuntu` |
| `ansible_password` | SSH password (prefer Vault) | `ansible_password=secret` |
| `ansible_ssh_private_key_file` | SSH key path | `~/.ssh/id_rsa` |
| `ansible_connection` | Connection type | `ssh`, `local`, `winrm`, `docker` |
| `ansible_become` | Use privilege escalation | `true` |
| `ansible_become_user` | User to become | `root` |
| `ansible_python_interpreter` | Python path on the target | `/usr/bin/python3` |

**Example using an alias:**

```ini
[webservers]
web-prod-1 ansible_host=10.0.1.15 ansible_user=ubuntu
web-prod-2 ansible_host=10.0.1.16 ansible_user=ubuntu
```

**Example for localhost:**

```ini
localhost ansible_connection=local
```

**Example for Windows hosts:**

```ini
[windows]
win1.example.com

[windows:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_port=5986
```

---

## 5. Organizing Variables: `group_vars` and `host_vars`

Instead of cramming variables into the inventory file, store them in directories alongside it. Ansible loads them automatically.

```
project/
├── inventory.ini
├── group_vars/
│   ├── all.yml              # applies to every host
│   ├── webservers.yml       # applies to the webservers group
│   └── dbservers.yml
├── host_vars/
│   ├── web1.example.com.yml # applies to one host
│   └── db1.example.com.yml
└── site.yml
```

**`group_vars/webservers.yml`**
```yaml
http_port: 80
max_clients: 200
```

**`host_vars/db1.example.com.yml`**
```yaml
db_role: primary
backup_enabled: true
```

**Variable precedence (simplified, low to high):** `all` group vars → parent group vars → child group vars → host vars → playbook/extra vars (`-e`).

---

## 6. Dynamic Inventory

A dynamic inventory builds the host list at runtime by querying an external source, so it always matches reality. This is essential when servers are created and destroyed automatically.

### 6.1 Inventory plugins (recommended)

Plugins are configured with a YAML file whose name ends in the plugin's required suffix (such as `aws_ec2.yml`).

**AWS EC2 example: `aws_ec2.yml`**
```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: production
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
hostnames:
  - private-ip-address
compose:
  ansible_host: private_ip_address
```

**Azure example: `azure_rm.yml`**
```yaml
plugin: azure.azcollection.azure_rm
include_vm_resource_groups:
  - rg-prod
auth_source: auto
conditional_groups:
  linux: "'linux' in os_profile.system"
  windows: "'windows' in os_profile.system"
keyed_groups:
  - key: tags.role
    prefix: role
```

**GCP example: `gcp_compute.yml`**
```yaml
plugin: google.cloud.gcp_compute
projects:
  - my-gcp-project
auth_kind: serviceaccount
service_account_file: ~/gcp-key.json
keyed_groups:
  - key: labels.env
    prefix: env
```

**Using and testing it:**

```bash
# Install the collection
ansible-galaxy collection install amazon.aws

# See what Ansible discovered
ansible-inventory -i aws_ec2.yml --list
ansible-inventory -i aws_ec2.yml --graph

# Run against it
ansible-playbook -i aws_ec2.yml site.yml
```

### 6.2 Inventory scripts (legacy approach)

Any executable that prints JSON when called with `--list` can act as an inventory. This is useful for custom sources.

```python
#!/usr/bin/env python3
import json

inventory = {
    "webservers": {
        "hosts": ["web1.example.com", "web2.example.com"],
        "vars": {"http_port": 80}
    },
    "dbservers": {
        "hosts": ["db1.example.com"]
    },
    "_meta": {
        "hostvars": {
            "web1.example.com": {"ansible_user": "ubuntu"}
        }
    }
}
print(json.dumps(inventory))
```

```bash
chmod +x my_inventory.py
ansible-playbook -i my_inventory.py site.yml
```

### 6.3 Other common sources

VMware vSphere, OpenStack, Kubernetes, Docker, Terraform state, ServiceNow CMDB, NetBox, Red Hat Satellite.

---

## 7. Inventory Directory (Mixed Sources)

Point Ansible at a directory and it merges every inventory file/plugin inside.

```
inventories/
├── production/
│   ├── hosts.ini            # static on-prem hosts
│   ├── aws_ec2.yml          # dynamic cloud hosts
│   ├── group_vars/
│   │   └── all.yml
│   └── host_vars/
└── staging/
    ├── hosts.ini
    └── group_vars/
        └── all.yml
```

```bash
ansible-playbook -i inventories/production site.yml
ansible-playbook -i inventories/staging site.yml
```

Multiple inventories can also be passed together:

```bash
ansible-playbook -i hosts.ini -i aws_ec2.yml site.yml
```

---

## 8. Inline / Ad-hoc Inventory

```bash
# Note the trailing comma, required for a single host
ansible all -i "192.168.1.10," -m ping -u ubuntu

# Multiple hosts
ansible all -i "10.0.0.5,10.0.0.6," -m ping
```

---

## 9. Targeting Hosts with Patterns

Once an inventory exists, patterns choose which hosts to act on.

| Pattern | Meaning |
|---|---|
| `all` or `*` | All hosts |
| `webservers` | One group |
| `webservers:dbservers` | Union (either group) |
| `webservers:&production` | Intersection (in both) |
| `webservers:!staging` | Exclusion (webservers not in staging) |
| `web*.example.com` | Wildcard |
| `webservers[0]` | First host in the group |
| `webservers[0:2]` | Slice of hosts |
| `--limit web1.example.com` | Restrict a run to specific hosts |

```bash
ansible 'webservers:!staging' -m ping
ansible-playbook site.yml --limit "web1.example.com,web2.example.com"
```

---

## 10. Useful Inventory Commands

```bash
# Show full inventory as JSON
ansible-inventory -i inventory.ini --list

# Show a tree of groups and hosts
ansible-inventory -i inventory.ini --graph

# Show variables for one host
ansible-inventory -i inventory.ini --host web1.example.com

# Export as YAML
ansible-inventory -i inventory.ini --list --yaml

# Test connectivity to everything
ansible all -i inventory.ini -m ping

# List the hosts a playbook would hit
ansible-playbook -i inventory.ini site.yml --list-hosts
```

---

## 11. Using Inventory in a Playbook

```yaml
# site.yml
- name: Configure web servers
  hosts: webservers
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Show a group variable
      ansible.builtin.debug:
        msg: "Listening on port {{ http_port }}"

- name: Configure databases
  hosts: dbservers
  become: true
  tasks:
    - name: Install PostgreSQL
      ansible.builtin.package:
        name: postgresql
        state: present
```

**Accessing other hosts' data (magic variables):**

```yaml
- debug:
    msg: "Group members: {{ groups['webservers'] }}"

- debug:
    msg: "This host is in groups: {{ group_names }}"

- debug:
    msg: "DB IP is {{ hostvars['db1.example.com']['ansible_host'] }}"
```

---

## 12. Static vs. Dynamic: Comparison

| Aspect | Static | Dynamic |
|---|---|---|
| Source of truth | File you maintain | External system (cloud, CMDB) |
| Updates | Manual | Automatic at run time |
| Setup effort | Very low | Moderate (credentials, plugin) |
| Handles autoscaling | No | Yes |
| Risk of stale data | High | Low |
| Best for | Small or fixed infrastructure | Cloud and large or changing infrastructure |

---

## 13. Real-World Use Cases

### 13.1 Multi-tier application deployment
Group servers by role (`webservers`, `appservers`, `dbservers`) and run different roles on each group in one playbook. Deploy the app tier only after the database tier is configured.

### 13.2 Multiple environments (dev / staging / prod)
Keep a separate inventory per environment with its own `group_vars`. The same playbook runs everywhere:

```bash
ansible-playbook -i inventories/staging site.yml
ansible-playbook -i inventories/production site.yml
```

### 13.3 Cloud autoscaling fleets
Use `aws_ec2`, `azure_rm`, or `gcp_compute` plugins with tag-based grouping so newly launched instances are configured automatically, with no manual host list.

### 13.4 Rolling updates and patching
Target a subset of hosts using patterns or `serial` to patch a few servers at a time and avoid downtime:

```yaml
- hosts: webservers
  serial: 2
  tasks:
    - name: Update packages
      ansible.builtin.package:
        name: "*"
        state: latest
```

### 13.5 Mixed OS environments
Separate `linux` and `windows` groups with different connection settings (`ssh` vs `winrm`) in the same inventory.

### 13.6 Network automation
Group routers and switches by vendor or role and set `ansible_connection=network_cli` and `ansible_network_os` as group variables.

### 13.7 Hybrid infrastructure
Combine on-prem static hosts and cloud dynamic hosts in one inventory directory and manage them with a single playbook.

### 13.8 Geographic or compliance segmentation
Create groups like `region_eu` and `region_us` to apply region-specific configuration or compliance rules.

---

## 14. Best Practices

- **Use groups by function and by environment**, and combine them with `:children`.
- **Prefer YAML or dynamic plugins** over legacy scripts.
- **Keep variables out of the inventory file**; use `group_vars/` and `host_vars/`.
- **Never store plain-text secrets** in inventory. Use **Ansible Vault** or an external secrets manager.
- **Separate inventories per environment** to prevent accidentally running prod changes against staging (or the reverse).
- **Use meaningful aliases** with `ansible_host` for IPs.
- **Validate before running:** `ansible-inventory --graph` and `--list-hosts`.
- **Store inventory in version control** (excluding secrets).
- **Use `--limit` and `--check`** when testing changes on production.
- **Tag cloud resources consistently** so dynamic grouping works predictably.

---

## 15. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `skipping: no hosts matched` | Pattern or group name wrong | Run `ansible-inventory --graph` and check spelling |
| `UNREACHABLE` | Wrong IP, user, key, or port | Verify `ansible_host`, `ansible_user`, SSH key; test with `ssh` manually |
| `Permission denied (publickey)` | Wrong key or user | Check `ansible_ssh_private_key_file` and the remote `authorized_keys` |
| Dynamic inventory returns empty | Wrong filters, credentials, or region | Run `ansible-inventory -i plugin.yml --list -vvv` |
| Variable not applied | Precedence or wrong group name | Run `ansible-inventory --host <name>` to see merged vars |
| Plugin not found | Collection missing | `ansible-galaxy collection install <collection>` |
| Python not found on target | Interpreter path differs | Set `ansible_python_interpreter=/usr/bin/python3` |

---

## 16. Quick Summary

- The **inventory** tells Ansible *which hosts* to manage and *how to reach them*.
- **Static** inventories (INI/YAML) suit small, stable setups; **dynamic** inventories (cloud plugins/scripts) suit changing infrastructure.
- Use **groups**, **group_vars**, and **host_vars** to keep configuration organized.
- Use **patterns** and `--limit` to target exactly the hosts you want.
- Keep separate inventories per environment and protect secrets with Vault.

---

## References

- Ansible inventory guide: https://docs.ansible.com/ansible/latest/inventory_guide/index.html
- Inventory plugins: https://docs.ansible.com/ansible/latest/plugins/inventory.html
- Patterns: https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html
- Ansible Vault: https://docs.ansible.com/ansible/latest/vault_guide/index.html
