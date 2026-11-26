# Ansible Collections

## 1. What are Ansible Collections?

Ansible Collections are packages used to distribute and manage Ansible content.

A collection can contain:

- Modules – Perform tasks such as installing packages, creating users, and managing services.
- Plugins – Extend Ansible functionality.
- Roles – Reusable automation tasks.
- Playbooks – Automation examples.
- Documentation – Instructions for using the collection.

**Example:** Instead of using built-in modules for everything, you can install collections to manage AWS, Azure, Kubernetes, Docker, and other technologies.

## 2. Collection Naming Convention

Ansible collections follow this naming format:

```text
namespace.collection_name
```

Example:

```text
community.general
ansible.posix
amazon.aws
kubernetes.core
```

| Collection | Purpose |
|---|---|
| `ansible.posix` | Linux and POSIX system administration |
| `community.general` | General-purpose automation modules |
| `amazon.aws` | AWS resource management |
| `azure.azcollection` | Azure resource management |
| `kubernetes.core` | Kubernetes automation |
| `community.docker` | Docker management |

## 3. Install Ansible Collections

Use the `ansible-galaxy` command to install collections.

Install a general collection:

```bash
ansible-galaxy collection install community.general
```

Install AWS collection:

```bash
ansible-galaxy collection install amazon.aws
```

Install Kubernetes collection:

```bash
ansible-galaxy collection install kubernetes.core
```

Install a specific version:

```bash
ansible-galaxy collection install community.general:8.6.0
```

## 4. List Installed Collections

Check installed collections:

```bash
ansible-galaxy collection list
```

Check a specific collection:

```bash
ansible-galaxy collection list community.general
```

## 5. Collection Directory Structure

By default, collections are installed under:

```text
~/.ansible/collections/ansible_collections/
```

Example:

```text
~/.ansible/
└── collections/
    └── ansible_collections/
        ├── community/
        │   └── general/
        ├── amazon/
        │   └── aws/
        └── kubernetes/
            └── core/
```

## 6. Using Collections in Playbooks

You can use a collection in two ways.

### Method 1: Fully Qualified Collection Name (FQCN)

Specify the complete module name.

Example: Install Apache on Ubuntu.

```yaml
---
- name: Install Apache
  hosts: all
  become: true

  tasks:
    - name: Install Apache package
      ansible.builtin.apt:
        name: apache2
        state: present
        update_cache: true
```

Example using the `community.general` collection:

```yaml
---
- name: Manage system packages
  hosts: all
  become: true

  tasks:
    - name: Install a package
      community.general.apk:
        name: nginx
        state: present
```

Note: The APK module is intended for Alpine Linux.

### Method 2: Declare Collections at Playbook Level

Use the `collections` keyword.

```yaml
---
- name: Collection Example
  hosts: all
  become: true

  collections:
    - community.general

  tasks:
    - name: Install package
      apk:
        name: nginx
        state: present
```

The collection must be installed, and the module must be available in that collection.

## 7. Install Collections Using requirements.yml

For team projects, maintain collections in a requirements file.

Create a file:

```text
requirements.yml
```

Add the following:

```yaml
---
collections:
  - name: community.general
    version: ">=8.0.0"

  - name: ansible.posix

  - name: amazon.aws

  - name: kubernetes.core
```

Install all collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

This makes it easier to install the required collections on another Ansible control node.

## 8. Create Your Own Collection

Create a new collection:

```bash
ansible-galaxy collection init mycompany.linux
```

Directory structure:

```text
linux/
└── mycompany/
    └── linux/
        ├── README.md
        ├── galaxy.yml
        ├── plugins/
        │   ├── modules/
        │   ├── inventory/
        │   └── filter/
        ├── roles/
        ├── playbooks/
        └── docs/
```

Build the collection:

```bash
ansible-galaxy collection build
```

Install your collection:

```bash
ansible-galaxy collection install mycompany-linux-1.0.0.tar.gz
```

## 9. Useful Ansible Collection Commands

| Command | Purpose |
|---|---|
| `ansible-galaxy collection list` | List installed collections |
| `ansible-galaxy collection install community.general` | Install a collection |
| `ansible-galaxy collection download community.general` | Download a collection |
| `ansible-galaxy collection verify community.general` | Verify an installed collection |
| `ansible-galaxy collection init mycompany.linux` | Create a collection structure |
| `ansible-galaxy collection build` | Build a collection package |
| `ansible-galaxy collection install -r requirements.yml` | Install collections from a requirements file |

## 10. Collections vs Roles

| Feature | Collections | Roles |
|---|---|---|
| Purpose | Package Ansible automation content | Organize reusable automation tasks |
| Modules | Can contain modules | Cannot define custom modules as role content |
| Plugins | Can contain plugins | Cannot package plugins as collection content |
| Roles | Can contain multiple roles | A role is a reusable automation unit |
| Distribution | Ansible Galaxy | Ansible Galaxy |

**Important:** Collections can contain multiple roles, modules, plugins, and documentation.

## 11. Real-Time DevOps Example

For a DevOps project involving AWS, Docker, and Kubernetes, you might install:

```bash
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install community.docker
ansible-galaxy collection install kubernetes.core
```

Then use their modules in playbooks:

```yaml
amazon.aws.ec2_instance:
community.docker.docker_container:
kubernetes.core.k8s:
```

These modules help automate infrastructure provisioning, container management, and Kubernetes deployments.

---

## Summary

- Collections are packages of reusable Ansible automation content.
- Collections can contain modules, plugins, roles, and documentation.
- Use `ansible-galaxy collection install` to install collections.
- Use FQCN to identify modules uniquely.
- Use `requirements.yml` to manage collection dependencies.
- Collections make Ansible automation easier to reuse and share.
