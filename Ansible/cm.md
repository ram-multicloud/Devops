# Configuration Management (CM) Guide

## 1. What is CM?

**Configuration Management (CM)** is a way to set up, change, and maintain servers and software **using code instead of manual work**.

You write down how a system *should* look (installed packages, config files, running services). A CM tool then makes every server match that description automatically.

**Simple example:**

> "All web servers must have Nginx installed, use this config file, and be running."

Write it once, and the tool applies it to 1 server or 1,000 servers the same way.

---

## 2. Why do we need CM?

Without CM, admins log in to each server and change things by hand. This causes problems:

| Problem without CM | How CM solves it |
|---|---|
| Servers end up different over time (**configuration drift**) | Every server is forced to match the same definition |
| Manual work is slow and error-prone | Automation applies changes in minutes |
| Hard to rebuild a crashed server | Rebuild from code, exactly as before |
| No record of who changed what | All changes are stored in Git with history |
| "Works on my machine" bugs | Dev, test, and production are identical |
| Risky changes with no undo | Review changes first, roll back if needed |
| Hard to prove security compliance | The code is the proof of how systems are set up |

---

## 3. How CM works

### Basic flow

```
┌────────────────────┐
│ 1. Write config    │   (YAML / DSL files describing desired state)
│    as code         │
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ 2. Store in Git    │   (history, review, rollback)
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ 3. CM tool runs    │   (Ansible / Puppet / Chef / Salt)
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ 4. Servers are     │   (only needed changes are made)
│    configured      │
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ 5. Monitor & fix   │   (detect drift, correct it)
│    drift           │
└────────────────────┘
```

### What the tool does on each server

```
        Desired state                Actual state
   (what the code says)        (what the server has)
              │                          │
              └────────────┬─────────────┘
                           ▼
                    ┌─────────────┐
                    │   Compare   │
                    └──────┬──────┘
                           ▼
              ┌────────────────────────┐
              │ Same?  → do nothing    │
              │ Different? → fix it    │
              └────────────────────────┘
```

This "only change what is different" behavior is called **idempotency**. Running the same configuration many times always gives the same result.

---

## 4. Two ways CM tools work

### Push model (e.g., Ansible)

A central machine **pushes** the configuration to servers when you run it.

```
                ┌──────────────┐
                │ Control node │
                │ (your PC /   │
                │  Ansible)    │
                └──────┬───────┘
          push over SSH│
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │Server A │   │Server B │   │Server C │
   └─────────┘   └─────────┘   └─────────┘
   (no agent needed)
```

### Pull model (e.g., Puppet, Chef)

An **agent** on each server regularly **pulls** its configuration from a central server.

```
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │Server A │   │Server B │   │Server C │
   │ (agent) │   │ (agent) │   │ (agent) │
   └────┬────┘   └────┬────┘   └────┬────┘
        │ pull every  │ 30 min      │
        └─────────────┼─────────────┘
                      ▼
              ┌───────────────┐
              │ Central server│
              │ (Puppet/Chef) │
              └───────────────┘
```

| | Push | Pull |
|---|---|---|
| Who starts the change? | You | The agent, on a schedule |
| Agent needed? | Usually no | Yes |
| Drift correction | When you run it | Automatic and continuous |
| Setup | Simple | More setup |

---

## 5. Types of CM tools

### 5.1 Ansible

- **Language:** YAML (playbooks)
- **Model:** Push, **agentless** (uses SSH)
- **Strengths:** Easy to learn, quick to start, great for automation and orchestration
- **Weaknesses:** Slower on very large fleets; no automatic drift correction unless scheduled
- **Best for:** Beginners, small to medium environments, mixed tasks (deploy + configure)

```yaml
- hosts: webservers
  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present
    - name: Start nginx
      service:
        name: nginx
        state: started
```

### 5.2 Puppet

- **Language:** Puppet DSL (declarative)
- **Model:** Pull, agent-based (master/agent)
- **Strengths:** Mature, strong compliance and reporting, continuous drift correction
- **Weaknesses:** Steeper learning curve, needs agents and a master server
- **Best for:** Large enterprises with strict compliance needs

```puppet
package { 'nginx':
  ensure => installed,
}
service { 'nginx':
  ensure => running,
}
```

### 5.3 Chef

- **Language:** Ruby (recipes and cookbooks)
- **Model:** Pull, agent-based (Chef server + Chef client)
- **Strengths:** Very flexible because it uses a real programming language
- **Weaknesses:** Requires Ruby knowledge, more complex
- **Best for:** Developer-heavy teams with complex logic

```ruby
package 'nginx'
service 'nginx' do
  action [:enable, :start]
end
```

### 5.4 SaltStack (Salt)

- **Language:** YAML + Jinja templates
- **Model:** Push or pull; uses a fast message bus (master/minion)
- **Strengths:** Very fast at large scale, event-driven automation
- **Weaknesses:** More complex architecture
- **Best for:** Very large infrastructures needing speed

```yaml
nginx:
  pkg.installed: []
  service.running:
    - enable: True
```

### 5.5 Terraform / OpenTofu (infrastructure provisioning)

- **Language:** HCL
- **Role:** **Creates** infrastructure (cloud servers, networks, databases), rather than configuring what runs inside
- **Best for:** Cloud setup on AWS, Azure, GCP

> Terraform and Ansible are often used **together**: Terraform creates the servers, Ansible configures them.

### 5.6 Kubernetes and Docker (container-based configuration)

- **Docker:** Packages an app and its configuration into an image, so the environment is identical everywhere.
- **Kubernetes:** Uses YAML manifests to declare the desired state of apps, and constantly works to keep reality matching it.
- **Best for:** Modern, cloud-native applications.

---

## 6. Tool comparison

| Feature | Ansible | Puppet | Chef | Salt | Terraform |
|---|---|---|---|---|---|
| **Language** | YAML | Puppet DSL | Ruby | YAML + Jinja | HCL |
| **Model** | Push | Pull | Pull | Push / Pull | API-driven |
| **Agent needed** | No | Yes | Yes | Usually yes | No |
| **Learning curve** | Easy | Medium-hard | Hard | Medium | Medium |
| **Main use** | Configure + automate | Configure + compliance | Configure | Configure at scale | Provision infrastructure |

---

## 7. Where CM fits in a real workflow

```
 Terraform            Ansible / Puppet / Chef          Monitoring
┌──────────┐         ┌──────────────────────┐        ┌──────────────┐
│ Create   │ ──────▶ │ Install software,    │ ─────▶ │ Check health │
│ servers  │         │ apply configuration  │        │ & drift      │
└──────────┘         └──────────────────────┘        └──────┬───────┘
     ▲                                                      │
     └────────── Update code in Git when changes are needed ┘
```

---

## 8. Key terms

| Term | Meaning |
|---|---|
| **Desired state** | How the system is supposed to be |
| **Drift** | When a server no longer matches the desired state |
| **Idempotency** | Running the same config repeatedly gives the same result |
| **Infrastructure as Code (IaC)** | Managing infrastructure through code files |
| **Playbook / Manifest / Recipe** | The file that describes the configuration |
| **Inventory** | The list of servers being managed |
| **Agent** | Small program on a server that applies configuration |

---

## 9. Summary

- **CM = managing systems with code** so they are consistent, repeatable, and tracked.
- **Why:** removes manual errors, prevents drift, saves time, and makes recovery easy.
- **How:** write desired state → store in Git → tool applies → monitor and fix drift.
- **Ansible:** easiest, agentless, push. **Puppet / Chef:** agent-based, pull, continuous enforcement. **Salt:** fast at scale. **Terraform:** creates infrastructure.
- **Tip:** start with Ansible to learn the ideas, then explore the others as needs grow.
