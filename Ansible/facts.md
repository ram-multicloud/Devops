
# Part 1: Ansible Facts

## 1. What are facts?

Facts are pieces of information Ansible **automatically discovers about each managed host** at the start of a play: OS, version, IP addresses, CPU, memory, disks, hostname, and so on.

They are collected by the `setup` module, which runs implicitly as the first task (`Gathering Facts`) whenever `gather_facts` is true (the default).

Why they matter: facts let **one playbook or role behave differently per host**, for example `apt` on Ubuntu and `dnf` on RHEL, without writing separate playbooks.

---

## 2. See all facts of a host

```bash
# Every fact (large JSON output)
ansible all -i inventory.ini -m ansible.builtin.setup

# One host only
ansible web1 -i inventory.ini -m ansible.builtin.setup

# Filter by name pattern
ansible web1 -i inventory.ini -m ansible.builtin.setup -a "filter=ansible_distribution*"
ansible web1 -i inventory.ini -m ansible.builtin.setup -a "filter=ansible_memtotal_mb"

# Only a subset (faster)
ansible web1 -i inventory.ini -m ansible.builtin.setup -a "gather_subset=network,!hardware"
```

Print facts inside a playbook:

```yaml
- hosts: all
  tasks:
    - name: Show all facts
      ansible.builtin.debug:
        var: ansible_facts
```

---

## 3. Most useful facts

| Fact | Example value | Typical use |
|---|---|---|
| `ansible_os_family` | `Debian`, `RedHat`, `Suse` | Choose package names, paths, modules |
| `ansible_distribution` | `Ubuntu`, `CentOS`, `Rocky`, `Amazon` | Distro-specific logic |
| `ansible_distribution_version` | `22.04`, `9.3` | Version checks |
| `ansible_distribution_major_version` | `22`, `9` | Major-version branching |
| `ansible_distribution_release` | `jammy` | Debian-style codename |
| `ansible_pkg_mgr` | `apt`, `dnf`, `yum` | Detect package manager |
| `ansible_service_mgr` | `systemd` | Detect init system |
| `ansible_architecture` | `x86_64`, `aarch64` | Download correct binary |
| `ansible_hostname` | `web1` | Short hostname |
| `ansible_fqdn` | `web1.example.com` | Full hostname |
| `ansible_default_ipv4.address` | `10.0.0.12` | Host's main IP |
| `ansible_processor_vcpus` | `4` | Tune thread/worker counts |
| `ansible_memtotal_mb` | `7821` | Tune JVM heap |
| `ansible_kernel` | `5.15.0-91-generic` | Kernel checks |
| `ansible_selinux.status` | `enabled` / `disabled` | SELinux handling on RedHat |
| `ansible_mounts` | list of mounts | Disk space checks |
| `ansible_env.HOME` | `/root` | Environment of the connecting user |

Two ways to refer to a fact (both work):

```yaml
"{{ ansible_os_family }}"            # classic injected variable
"{{ ansible_facts['os_family'] }}"   # recommended, namespaced, no "ansible_" prefix
```

---

## 4. Using facts: examples

### 4.1 Conditionals with `when`

```yaml
- name: Install Apache on Debian family
  ansible.builtin.apt:
    name: apache2
    state: present
  when: ansible_facts['os_family'] == "Debian"

- name: Install Apache on RedHat family
  ansible.builtin.dnf:
    name: httpd
    state: present
  when: ansible_facts['os_family'] == "RedHat"
```

### 4.2 Distro-independent package module

```yaml
- name: Install packages (works on apt, dnf and yum)
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop:
    - curl
    - unzip
```

Package **names** often differ per OS (`apache2` vs `httpd`), which is why we pair this with per-OS variable files (see Section 4.5).

### 4.3 Facts in templates (Jinja2)

`templates/motd.j2`:

```jinja
Welcome to {{ ansible_facts['hostname'] }}
OS      : {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}
IP      : {{ ansible_facts['default_ipv4']['address'] }}
CPU     : {{ ansible_facts['processor_vcpus'] }} vCPU
Memory  : {{ ansible_facts['memtotal_mb'] }} MB
```

```yaml
- name: Deploy MOTD
  ansible.builtin.template:
    src: motd.j2
    dest: /etc/motd
    mode: "0644"
```

### 4.4 Dynamic tuning with facts (JVM heap = 50% of RAM)

```yaml
- name: Compute JVM heap
  ansible.builtin.set_fact:
    jvm_heap_mb: "{{ (ansible_facts['memtotal_mb'] * 0.5) | int }}"

- name: Show heap
  ansible.builtin.debug:
    msg: "Xmx will be {{ jvm_heap_mb }}m"
```

### 4.5 Load a variables file per OS (the key pattern for multi-OS roles)

```
vars/
  Debian.yml
  RedHat.yml
  default.yml
```

```yaml
- name: Load OS specific variables
  ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files:
        - "{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
        - "{{ ansible_facts['distribution'] }}.yml"
        - "{{ ansible_facts['os_family'] }}.yml"
        - default.yml
      paths:
        - vars
```

`first_found` picks the **most specific** file that exists, so `Rocky-9.yml` beats `RedHat.yml`, which beats `default.yml`.

### 4.6 Group hosts by OS dynamically

```yaml
- hosts: all
  tasks:
    - name: Group by OS family
      ansible.builtin.group_by:
        key: "os_{{ ansible_facts['os_family'] }}"

- hosts: os_Debian
  tasks:
    - ansible.builtin.debug: msg="Debian-family host"

- hosts: os_RedHat
  tasks:
    - ansible.builtin.debug: msg="RedHat-family host"
```

### 4.7 Version comparison

```yaml
- name: Only on RHEL/Rocky 9 or newer
  ansible.builtin.debug:
    msg: "Modern RedHat"
  when:
    - ansible_facts['os_family'] == "RedHat"
    - ansible_facts['distribution_major_version'] | int >= 9
```

### 4.8 Facts about other hosts

```yaml
- hosts: all
  tasks:
    - ansible.builtin.debug:
        msg: "{{ hostvars['db1']['ansible_facts']['default_ipv4']['address'] }}"
```

Works only if `db1` facts were already gathered in the same run.

---

## 5. Custom (local) facts

Put `.fact` files on the **managed host** in `/etc/ansible/facts.d/`. They can be INI, JSON, or an executable that prints JSON. They appear under `ansible_local`.

`/etc/ansible/facts.d/app.fact`:

```ini
[general]
owner=devops-team
env=staging
```

```yaml
- ansible.builtin.debug:
    msg: "Env is {{ ansible_local['app']['general']['env'] }}"
```

Create it with Ansible:

```yaml
- name: Create facts.d directory
  ansible.builtin.file:
    path: /etc/ansible/facts.d
    state: directory
    mode: "0755"

- name: Add custom fact
  ansible.builtin.copy:
    dest: /etc/ansible/facts.d/app.fact
    content: |
      [general]
      owner=devops-team
      env=staging
    mode: "0644"

- name: Re-read facts
  ansible.builtin.setup:
    filter: ansible_local
```

---

## 6. `set_fact`: create your own facts at runtime

```yaml
- name: Detect Tomcat install state
  ansible.builtin.stat:
    path: /opt/tomcat/bin/catalina.sh
  register: tomcat_bin

- name: Remember result
  ansible.builtin.set_fact:
    tomcat_installed: "{{ tomcat_bin.stat.exists }}"
    cacheable: true     # optional, survives with fact caching
```

---

## 7. Special fact-gathering modules

```yaml
- name: Gather service facts
  ansible.builtin.service_facts:

- ansible.builtin.debug:
    msg: "firewalld is {{ ansible_facts.services['firewalld.service'].state | default('not installed') }}"
```

```yaml
- name: Gather package facts
  ansible.builtin.package_facts:
    manager: auto

- ansible.builtin.debug:
    msg: "Java installed"
  when: "'java-17-openjdk' in ansible_facts.packages or 'openjdk-17-jdk' in ansible_facts.packages"
```

---

## 8. Controlling fact gathering

```yaml
# Skip (faster) when you don't need facts
- hosts: all
  gather_facts: false

# Gather only what you need
- hosts: all
  gather_facts: true
  gather_subset:
    - "!all"
    - "!min"
    - network
```

Gather facts later, on demand:

```yaml
- hosts: all
  gather_facts: false
  tasks:
    - ansible.builtin.setup:
        gather_subset: min
```

If you use `gather_facts: false`, anything using `ansible_os_family` etc. will fail. Gather first.

Fact caching in `ansible.cfg`:

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400
```

---

## 9. Common mistakes

| Mistake | Fix |
|---|---|
| Comparing `ansible_os_family == "redhat"` | Value is case-sensitive: `"RedHat"` |
| Using `ansible_distribution == "RedHat"` for CentOS/Rocky | Use `os_family` for the whole family |
| `distribution_major_version >= 9` fails | It is a string; use `\| int >= 9` |
| Facts undefined | `gather_facts: false` was set |
| Hard-coded `/usr/lib/jvm/...amd64` | Use `ansible_architecture` (see Part 2) |
| Using `command: yum/apt` | Use `package`, `apt`, or `dnf` modules |

---

# Part 2: Tomcat Role for Debian **and** RedHat

> **Note:** I could not open `https://github.com/direct-devops/ansible-roles-2026/tree/main/tomcat` (GitHub blocks automated access), so I wrote a complete, working role in the standard layout. Compare it with your repo and merge: the important parts are `vars/<OS>.yml`, the `include_vars` task, and the `package` / user / firewall / SELinux handling.
>
> If you paste your existing `tasks/main.yml`, `defaults/main.yml` and templates here, I can convert *your* files directly instead.

## 1. Role layout

```
roles/tomcat/
├── defaults/main.yml
├── handlers/main.yml
├── meta/main.yml
├── tasks/
│   ├── main.yml
│   ├── install.yml
│   └── configure.yml
├── templates/
│   ├── tomcat.service.j2
│   └── setenv.sh.j2
└── vars/
    ├── Debian.yml
    └── RedHat.yml
```

## 2. `defaults/main.yml`

```yaml
---
# Tomcat version and download location
# Check available versions at https://archive.apache.org/dist/tomcat/tomcat-10/
tomcat_major_version: 10
tomcat_version: "10.1.34"
tomcat_mirror: "https://archive.apache.org/dist/tomcat"
tomcat_download_url: "{{ tomcat_mirror }}/tomcat-{{ tomcat_major_version }}/v{{ tomcat_version }}/bin/apache-tomcat-{{ tomcat_version }}.tar.gz"

# Java
java_version: 17

# Install layout
tomcat_user: tomcat
tomcat_group: tomcat
tomcat_install_dir: /opt/tomcat
tomcat_home: "{{ tomcat_install_dir }}/apache-tomcat-{{ tomcat_version }}"
tomcat_current: "{{ tomcat_install_dir }}/current"

# Network
tomcat_http_port: 8080
tomcat_shutdown_port: 8005

# JVM (heap defaults to 50% of RAM, minimum 256 MB; override as needed)
tomcat_jvm_heap_mb: "{{ [((ansible_facts['memtotal_mb'] | int) * 0.5) | int, 256] | max }}"

# Firewall handling (RedHat: firewalld, Debian: ufw)
tomcat_manage_firewall: false
```

## 3. `vars/Debian.yml`

```yaml
---
tomcat_required_packages:
  - "openjdk-{{ java_version }}-jdk"
  - curl
  - tar
  - acl            # needed for become_user on Debian/Ubuntu

# Debian puts the JDK in an architecture specific folder
tomcat_java_home: "/usr/lib/jvm/java-{{ java_version }}-openjdk-{{ 'arm64' if ansible_facts['architecture'] == 'aarch64' else 'amd64' }}"

tomcat_nologin_shell: /usr/sbin/nologin
```

## 4. `vars/RedHat.yml`

```yaml
---
tomcat_required_packages:
  - "java-{{ java_version }}-openjdk-devel"
  - curl
  - tar

# RedHat family provides an architecture neutral symlink
tomcat_java_home: "/usr/lib/jvm/jre-{{ java_version }}-openjdk"

tomcat_nologin_shell: /sbin/nologin
```

## 5. `tasks/main.yml`

```yaml
---
- name: Load OS specific variables
  ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', params) }}"
  vars:
    params:
      files:
        - "{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
        - "{{ ansible_facts['distribution'] }}.yml"
        - "{{ ansible_facts['os_family'] }}.yml"
      paths:
        - vars
  tags: [tomcat, always]

- name: Fail early on unsupported OS family
  ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] in ['Debian', 'RedHat']
    fail_msg: "Unsupported OS family: {{ ansible_facts['os_family'] }}"
  tags: [tomcat, always]

- name: Install Tomcat
  ansible.builtin.import_tasks: install.yml
  tags: [tomcat, install]

- name: Configure Tomcat
  ansible.builtin.import_tasks: configure.yml
  tags: [tomcat, configure]
```

## 6. `tasks/install.yml`

```yaml
---
- name: Refresh package cache (Debian family)
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
  when: ansible_facts['os_family'] == "Debian"

- name: Install Java and required packages
  ansible.builtin.package:
    name: "{{ tomcat_required_packages }}"
    state: present

- name: Create tomcat group
  ansible.builtin.group:
    name: "{{ tomcat_group }}"
    system: true

- name: Create tomcat user
  ansible.builtin.user:
    name: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    system: true
    shell: "{{ tomcat_nologin_shell }}"
    home: "{{ tomcat_install_dir }}"
    create_home: false

- name: Create install directory
  ansible.builtin.file:
    path: "{{ tomcat_install_dir }}"
    state: directory
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    mode: "0755"

- name: Check if this Tomcat version is already extracted
  ansible.builtin.stat:
    path: "{{ tomcat_home }}/bin/catalina.sh"
  register: tomcat_binary

- name: Download and extract Tomcat
  ansible.builtin.unarchive:
    src: "{{ tomcat_download_url }}"
    dest: "{{ tomcat_install_dir }}"
    remote_src: true
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
  when: not tomcat_binary.stat.exists
  notify: Restart tomcat

- name: Point 'current' symlink to this version
  ansible.builtin.file:
    src: "{{ tomcat_home }}"
    dest: "{{ tomcat_current }}"
    state: link
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
  notify: Restart tomcat

- name: Make scripts executable
  ansible.builtin.file:
    path: "{{ tomcat_home }}/bin"
    state: directory
    mode: "u+rwX,g+rX,o+rX"
    recurse: true
```

## 7. `tasks/configure.yml`

```yaml
---
- name: Deploy setenv.sh (JVM options)
  ansible.builtin.template:
    src: setenv.sh.j2
    dest: "{{ tomcat_home }}/bin/setenv.sh"
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    mode: "0750"
  notify: Restart tomcat

- name: Deploy systemd unit
  ansible.builtin.template:
    src: tomcat.service.j2
    dest: /etc/systemd/system/tomcat.service
    owner: root
    group: root
    mode: "0644"
  notify:
    - Reload systemd
    - Restart tomcat

# ---- RedHat only: firewalld ----
- name: Open Tomcat port in firewalld (RedHat family)
  ansible.posix.firewalld:
    port: "{{ tomcat_http_port }}/tcp"
    permanent: true
    immediate: true
    state: enabled
  when:
    - tomcat_manage_firewall | bool
    - ansible_facts['os_family'] == "RedHat"

# ---- Debian only: ufw ----
- name: Open Tomcat port in ufw (Debian family)
  community.general.ufw:
    rule: allow
    port: "{{ tomcat_http_port | string }}"
    proto: tcp
  when:
    - tomcat_manage_firewall | bool
    - ansible_facts['os_family'] == "Debian"

# ---- RedHat only: SELinux ----
- name: Allow Tomcat port in SELinux (if non-default port)
  community.general.seport:
    ports: "{{ tomcat_http_port }}"
    proto: tcp
    setype: http_port_t
    state: present
  when:
    - ansible_facts['os_family'] == "RedHat"
    - ansible_facts['selinux']['status'] | default('disabled') == 'enabled'
    - tomcat_http_port | int != 8080

- name: Enable and start Tomcat
  ansible.builtin.systemd:
    name: tomcat
    enabled: true
    state: started
    daemon_reload: true
```

> `community.general.seport` needs `policycoreutils-python-utils` on RedHat. Add it to `tomcat_required_packages` in `vars/RedHat.yml` if you use a non-default port with SELinux enforcing.

## 8. `templates/tomcat.service.j2`

```jinja
# Managed by Ansible - host: {{ ansible_facts['hostname'] }} ({{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }})
[Unit]
Description=Apache Tomcat {{ tomcat_version }}
After=network.target

[Service]
Type=forking
User={{ tomcat_user }}
Group={{ tomcat_group }}

Environment="JAVA_HOME={{ tomcat_java_home }}"
Environment="CATALINA_HOME={{ tomcat_current }}"
Environment="CATALINA_BASE={{ tomcat_current }}"
Environment="CATALINA_PID={{ tomcat_current }}/temp/tomcat.pid"

ExecStart={{ tomcat_current }}/bin/startup.sh
ExecStop={{ tomcat_current }}/bin/shutdown.sh
Restart=on-failure
RestartSec=10
UMask=0027

[Install]
WantedBy=multi-user.target
```

## 9. `templates/setenv.sh.j2`

```jinja
#!/bin/sh
# Managed by Ansible
export JAVA_HOME="{{ tomcat_java_home }}"
export CATALINA_OPTS="$CATALINA_OPTS -Xms{{ tomcat_jvm_heap_mb }}m -Xmx{{ tomcat_jvm_heap_mb }}m"
export CATALINA_OPTS="$CATALINA_OPTS -Djava.security.egd=file:/dev/./urandom"
```

## 10. `handlers/main.yml`

```yaml
---
- name: Reload systemd
  ansible.builtin.systemd:
    daemon_reload: true

- name: Restart tomcat
  ansible.builtin.systemd:
    name: tomcat
    state: restarted
```

## 11. `meta/main.yml`

```yaml
---
galaxy_info:
  role_name: tomcat
  author: your-name
  description: Install and configure Apache Tomcat on Debian and RedHat families
  license: MIT
  min_ansible_version: "2.12"
  platforms:
    - name: Ubuntu
      versions: [focal, jammy, noble]
    - name: Debian
      versions: [bullseye, bookworm]
    - name: EL
      versions: ["8", "9"]
    - name: Fedora
      versions: [all]
  galaxy_tags: [tomcat, java, web]
dependencies: []
```

## 12. Required collections

```yaml
# requirements.yml
collections:
  - name: ansible.posix
  - name: community.general
```

```bash
ansible-galaxy collection install -r requirements.yml
```

## 13. Example playbook and inventory

`inventory.ini`:

```ini
[tomcat_servers]
ubuntu1 ansible_host=10.0.0.11
rocky1  ansible_host=10.0.0.12

[tomcat_servers:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/mykey.pem
```

`site.yml`:

```yaml
---
- name: Install Tomcat on Debian and RedHat servers
  hosts: tomcat_servers
  become: true
  gather_facts: true      # required: the role uses ansible_facts
  roles:
    - role: tomcat
      vars:
        tomcat_http_port: 8080
        tomcat_manage_firewall: true
```

Run:

```bash
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --tags install
ansible-playbook -i inventory.ini site.yml --check --diff
```

Verify on each host:

```bash
systemctl status tomcat
curl -I http://localhost:8080
```

---

## 14. What changed to support both OS families (summary)

| Concern | Debian family | RedHat family | How the role handles it |
|---|---|---|---|
| Java package | `openjdk-17-jdk` | `java-17-openjdk-devel` | `vars/<os_family>.yml` |
| `JAVA_HOME` | `/usr/lib/jvm/java-17-openjdk-amd64` | `/usr/lib/jvm/jre-17-openjdk` | `vars/<os_family>.yml` plus `ansible_architecture` |
| Package install | apt | dnf/yum | `ansible.builtin.package` |
| apt cache | needed | n/a | `when: os_family == "Debian"` |
| nologin shell | `/usr/sbin/nologin` | `/sbin/nologin` | per-OS variable |
| Firewall | ufw | firewalld | separate tasks gated by `when` |
| SELinux | usually absent | usually enforcing | gated by `ansible_facts['selinux']` |
| Service manager | systemd | systemd | same unit template |

## 15. Quick testing tips

```bash
# Lint
ansible-lint roles/tomcat

# Dry run
ansible-playbook -i inventory.ini site.yml --check --diff

# Test on both families with containers (molecule)
pip install molecule molecule-plugins[docker]
# platforms: ubuntu:22.04, debian:12, rockylinux:9, almalinux:9
```

If you want a CI-style test matrix, use `geerlingguy/docker-ubuntu2204-ansible` and `geerlingguy/docker-rockylinux9-ansible` images, which include systemd.
