# Ansible Collections: Overview

## What is a collection?

A **collection** is a package that bundles Ansible content so it can be shared and reused. One collection can contain:

- Roles
- Modules
- Plugins (filters, lookups, inventory, callbacks, etc.)
- Playbooks
- Documentation

Think of it like a Python package, but for Ansible automation.

## Why use collections?

- **Reusable:** write once, use in many projects
- **Versioned:** pin an exact version so nothing breaks unexpectedly
- **Easy to share:** publish to Ansible Galaxy or a private server
- **Organized:** content is grouped under a clear name

## Naming

Every collection is named `namespace.collection_name`.

| Collection | Namespace | Name |
|------------|-----------|------|
| `ansible.posix` | ansible | posix |
| `community.general` | community | general |
| `amazon.aws` | amazon | aws |

Content inside it is called with the full name (FQCN):

```yaml
- name: Example
  ansible.posix.firewalld:
    service: http
    state: enabled
    permanent: true
```

## Folder structure

```
namespace/collection_name/
├── galaxy.yml        # metadata (name, version, author)
├── README.md
├── meta/
│   └── runtime.yml
├── plugins/
│   ├── modules/
│   ├── filter/
│   └── lookup/
├── roles/
├── playbooks/
├── docs/
└── tests/
```

## Using collections

### Install one

```bash
ansible-galaxy collection install community.general
```

### Install a specific version

```bash
ansible-galaxy collection install community.general:==8.0.0
```

### Install many with `requirements.yml`

```yaml
---
collections:
  - name: community.general
    version: ">=8.0.0"
  - name: ansible.posix
```

```bash
ansible-galaxy collection install -r requirements.yml
```

### See what is installed

```bash
ansible-galaxy collection list
```

### Upgrade

```bash
ansible-galaxy collection install community.general --upgrade
```

## Where collections come from

| Source | Use for |
|--------|---------|
| Ansible Galaxy | Public, free sharing |
| Automation Hub | Private or certified content (Red Hat) |
| Git repository | Install directly from a repo or tag |
| Tarball (`.tar.gz`) | Offline or internal sharing |

## Collections vs roles

| | Role | Collection |
|---|------|------------|
| Contains | Tasks, templates, variables | Roles, modules, plugins, playbooks |
| Size | One job | Many related jobs |
| Custom modules | No | Yes |
| Versioned package | Limited | Yes |

Use a **role** for one task set (like installing nginx). Use a **collection** when you want to ship a group of related content together.

## Quick command list

| Task | Command |
|------|---------|
| Create | `ansible-galaxy collection init name.space` |
| Build | `ansible-galaxy collection build` |
| Publish | `ansible-galaxy collection publish file.tar.gz --token TOKEN` |
| Install | `ansible-galaxy collection install name.space` |
| List | `ansible-galaxy collection list` |

## Tips

- Always use the full name (FQCN) in playbooks.
- Pin versions in `requirements.yml` for stable runs.
- Never share your Galaxy API token publicly.
- Read a collection's docs with `ansible-doc namespace.collection.module_name`.
