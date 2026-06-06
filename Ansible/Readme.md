## 1. Configuration Management Overview

**Configuration Management (CM)** is the practice of deploying and managing configurations across remote systems in a consistent, repeatable way.

### Core Principle
> Configurations are **declarative** — you define *what* needs to be done, not *how* to do it.

### Popular CM Tools

| Tool | Type | Language |
|------|------|----------|
| **Ansible** | Push-based | YAML (Playbooks) |
| **Chef** | Pull-based | Ruby (Recipes) |
| **Puppet** | Pull-based | Puppet DSL |
| **Salt** | Push/Pull | YAML/Python |
| **PowerShell DSC** | Pull-based | PowerShell |

---

## 2. CM Approaches: Push vs Pull

### 🔵 Push-Based (e.g., Ansible)

The control node **pushes** configurations out to managed nodes.

**Requirements:**
- **Inventory** — a list of all target nodes (IP/hostname)
- **Credentials** — SSH keys or passwords to connect

```
Control Node  →  push config  →  Node 1
                              →  Node 2
                              →  Node 3
```

✅ **Pros:** No agent needed on remote nodes, simpler setup  
⚠️ **Cons:** Control node must have network access to all nodes

---

### 🟠 Pull-Based (e.g., Chef, Puppet)

Managed nodes **pull** configurations from a central server on a schedule.

**Requirements:**
- **Agent** installed on every managed node
- Agent periodically checks the server for new configurations

```
Chef/Puppet Server  ←  node pulls config  ←  Node 1 (agent)
                    ←  node pulls config  ←  Node 2 (agent)
```

✅ **Pros:** Scales well, nodes self-heal  
⚠️ **Cons:** Agent overhead, more complex setup

---

## 3. SSH Authentication

Linux machines support **two types** of authentication over SSH:

### 🔑 Type 1 — Username & Password

**How it works (plain English):**
1. Your machine initiates an SSH connection to the remote server
2. Server asks: "Who are you?"
3. You provide your username
4. Server asks: "Prove it — what's your password?"
5. You send the password (encrypted over the SSH tunnel)
6. Server verifies → access granted or denied

> ⚠️ Less secure. Vulnerable to brute-force attacks. Not recommended for production.

---

### 🗝️ Type 2 — Username & Key-Based

**How it works (plain English):**
1. You generate a **key pair** — a private key (stays on your machine) and a public key
2. The **public key** is copied to the remote server's `~/.ssh/authorized_keys`
3. When you connect, the server sends a challenge encrypted with your public key
4. Only your **private key** can decrypt it → proves your identity without sending a password

**Key Types:**
| Key | Location | Share? |
|-----|----------|--------|
| **Private Key** (`id_rsa`) | Your local machine | ❌ Never |
| **Public Key** (`id_rsa.pub`) | Remote server | ✅ Yes |

> ✅ Much more secure. Recommended for all server access and automation.

---

### 📄 The `known_hosts` File

When you first SSH into a server, a file called `~/.ssh/known_hosts` is created/updated.

**Purpose:**
- Stores the **fingerprint** (public key) of every server you've connected to
- On future connections, SSH checks if the server's fingerprint matches — protects against **Man-in-the-Middle (MITM)** attacks
- If it doesn't match, SSH warns you: `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`

