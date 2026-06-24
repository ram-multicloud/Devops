# Ansible Vault, Galaxy Roles & Fetch Module

---

## Ansible Vault

Ansible Vault encrypts sensitive data (passwords, keys, tokens) so they can be safely stored in version control.

### 1.1 Encrypt a File

```bash
ansible-vault encrypt secrets.yml
# prompts for vault password
```

### 1.2 Decrypt a File

```bash
ansible-vault decrypt secrets.yml
```

### 1.3 View Without Decrypting to Disk

```bash
ansible-vault view secrets.yml
```

### 1.4 Edit an Encrypted File

```bash
ansible-vault edit secrets.yml
```

### 1.5 Create a New Encrypted File

```bash
ansible-vault create secrets.yml
```

### 1.6 Re-key (Change Vault Password)

```bash
ansible-vault rekey secrets.yml
```

### 1.7 Encrypt a Single String (Inline Variable)

```bash
ansible-vault encrypt_string 'MyP@ssword' --name 'db_password'
```

Output (paste directly into a vars file or playbook):

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  61383937303566613132306338356665...
```

### 1.8 Store Vault Password in a File

```bash
echo "myvaultpassword" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

Run playbook using vault password file:

```bash
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

Or set it permanently in `ansible.cfg`:

```ini
[defaults]
vault_password_file = ~/.vault_pass
```

### 1.9 Sample Encrypted vars File

```yaml
# group_vars/all/secrets.yml  (encrypted with ansible-vault)
db_user: admin
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  61393765386566396162306263363434...
```

### 1.10 Using Vault Variables in a Playbook

```yaml
- name: Deploy App
  hosts: appservers
  vars_files:
    - group_vars/all/secrets.yml
  tasks:
    - name: Configure DB connection
      template:
        src: db.conf.j2
        dest: /etc/app/db.conf
```

### 1.11 Multiple Vault IDs (Ansible >= 2.4)

Useful when different teams own different secrets:

```bash
# Encrypt with a named vault ID
ansible-vault encrypt_string 'secret' --vault-id dev@~/.vault_dev --name 'api_key'

# Run playbook with multiple vault IDs
ansible-playbook site.yml \
  --vault-id dev@~/.vault_dev \
  --vault-id prod@~/.vault_prod
```

### Vault Quick-Reference

| Command | Purpose |
|---|---|
| `ansible-vault create` | Create new encrypted file |
| `ansible-vault encrypt` | Encrypt existing file |
| `ansible-vault decrypt` | Decrypt file to plaintext |
| `ansible-vault view` | View without writing to disk |
| `ansible-vault edit` | Edit in-place |
| `ansible-vault rekey` | Change vault password |
| `ansible-vault encrypt_string` | Encrypt inline variable |

---

## 2. Ansible Galaxy Roles

Ansible Galaxy is the public hub for sharing Ansible roles and collections.

### 2.1 Install a Role from Galaxy

```bash
ansible-galaxy role install geerlingguy.nginx
```

Installed to `~/.ansible/roles/` by default.

### 2.2 Install to a Custom Path

```bash
ansible-galaxy role install geerlingguy.nginx -p roles/
```

### 2.3 Install Specific Version

```bash
ansible-galaxy role install geerlingguy.nginx,3.2.0
```

### 2.4 Install from requirements.yml

Create `requirements.yml`:

```yaml
---
roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
  - name: geerlingguy.java
    version: "2.1.0"

collections:
  - name: community.general
    version: ">=7.0.0"
  - name: ansible.posix
```

Install all at once:

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

### 2.5 Create a New Role Skeleton

```bash
ansible-galaxy role init myrole
```

Generated structure:

```
myrole/
├── defaults/
│   └── main.yml        # default variables (lowest priority)
├── files/              # static files to copy
├── handlers/
│   └── main.yml        # handlers (e.g., restart nginx)
├── meta/
│   └── main.yml        # role metadata, dependencies
├── tasks/
│   └── main.yml        # main task list
├── templates/          # Jinja2 templates (.j2)
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml        # variables (higher priority than defaults)
```

### 2.6 Using a Role in a Playbook

```yaml
- name: Setup Web Server
  hosts: webservers
  roles:
    - geerlingguy.nginx
    - geerlingguy.java
```

### 2.7 Pass Variables to a Role

```yaml
- name: Setup Web Server
  hosts: webservers
  roles:
    - role: geerlingguy.nginx
      vars:
        nginx_listen_port: 8080
```

### 2.8 Include Role in Tasks (Dynamic)

```yaml
tasks:
  - name: Apply nginx role
    include_role:
      name: geerlingguy.nginx
```

### 2.9 Role Dependencies (meta/main.yml)

```yaml
# roles/myapp/meta/main.yml
dependencies:
  - role: geerlingguy.java
    vars:
      java_packages:
        - java-17-openjdk
  - role: geerlingguy.nginx
```

### 2.10 List Installed Roles

```bash
ansible-galaxy role list
```

### 2.11 Remove a Role

```bash
ansible-galaxy role remove geerlingguy.nginx
```

### 2.12 Install & Use a Collection

```bash
# Install collection
ansible-galaxy collection install community.general

# Use a module from the collection in a playbook
- name: Use community.general module
  community.general.ini_file:
    path: /etc/app.conf
    section: database
    option: host
    value: "localhost"
```

### Galaxy CLI Quick-Reference

| Command | Purpose |
|---|---|
| `ansible-galaxy role install <name>` | Install role |
| `ansible-galaxy role init <name>` | Scaffold new role |
| `ansible-galaxy role list` | List installed roles |
| `ansible-galaxy role remove <name>` | Remove role |
| `ansible-galaxy install -r requirements.yml` | Bulk install from file |
| `ansible-galaxy collection install <ns.col>` | Install collection |

---

## 3. Ansible Fetch — Copy Files from Remote to Control Node

The `fetch` module pulls files **from a remote managed node** to the **Ansible control node**. It is the reverse of the `copy` module.

### 3.1 Basic Fetch

```yaml
- name: Fetch /etc/hosts from remote
  ansible.builtin.fetch:
    src: /etc/hosts
    dest: /tmp/fetched/
```

Files are saved as:
```
/tmp/fetched/<hostname>/etc/hosts
```

### 3.2 Flat Fetch (No Host Sub-Directory)

```yaml
- name: Fetch app log
  ansible.builtin.fetch:
    src: /var/log/app/error.log
    dest: /tmp/logs/error.log
    flat: yes
```

> **Note:** With `flat: yes` and multiple hosts, each host overwrites the same destination file. Use a variable in the dest path if fetching from multiple hosts.

### 3.3 Fetch with Hostname in Destination

```yaml
- name: Fetch nginx access log per host
  ansible.builtin.fetch:
    src: /var/log/nginx/access.log
    dest: "/tmp/logs/{{ inventory_hostname }}_access.log"
    flat: yes
```

### 3.4 Fetch with Checksum Validation (default on)

By default, Ansible validates the file checksum after transfer. Disable only if needed:

```yaml
- name: Fetch without checksum
  ansible.builtin.fetch:
    src: /etc/app.conf
    dest: /tmp/configs/
    validate_checksum: no
```

### 3.5 Fail Gracefully if File Not Found

```yaml
- name: Fetch file if it exists
  ansible.builtin.fetch:
    src: /opt/app/config.yml
    dest: /tmp/fetched/
    fail_on_missing: no
```

### 3.6 Complete Example Playbook

```yaml
---
- name: Collect config and logs from app servers
  hosts: appservers
  tasks:

    - name: Fetch application config
      ansible.builtin.fetch:
        src: /etc/myapp/config.yml
        dest: "backups/configs/{{ inventory_hostname }}_config.yml"
        flat: yes

    - name: Fetch system journal (compressed)
      ansible.builtin.fetch:
        src: /var/log/syslog
        dest: "backups/logs/{{ inventory_hostname }}_syslog"
        flat: yes

    - name: Fetch nginx error log (skip if missing)
      ansible.builtin.fetch:
        src: /var/log/nginx/error.log
        dest: "backups/logs/{{ inventory_hostname }}_nginx_error.log"
        flat: yes
        fail_on_missing: no
```

Run:

```bash
ansible-playbook collect_configs.yml -i inventory.ini
```

### 3.7 Fetch vs Copy vs Synchronize

| Module | Direction | Use Case |
|---|---|---|
| `copy` | Control → Remote | Push a file to remote host |
| `fetch` | Remote → Control | Pull a file from remote host |
| `synchronize` | Both | Rsync-based bulk file sync |
| `slurp` | Remote → Control | Read file content into a variable (no disk write) |

### fetch Module Parameters

| Parameter | Default | Description |
|---|---|---|
| `src` | required | Absolute path on remote host |
| `dest` | required | Local path on control node |
| `flat` | `no` | Skip host sub-directory structure |
| `fail_on_missing` | `yes` | Fail if src file not found |
| `validate_checksum` | `yes` | Verify file integrity after fetch |

---

## Summary

| Topic | Key Command / Module |
|---|---|
| Ansible Vault | `ansible-vault encrypt/decrypt/view/edit` |
| Vault inline var | `ansible-vault encrypt_string` |
| Galaxy role install | `ansible-galaxy role install <name>` |
| Bulk install | `ansible-galaxy install -r requirements.yml` |
| Fetch from remote | `ansible.builtin.fetch` |
| Fetch reverse | `ansible.builtin.copy` (push to remote) |
