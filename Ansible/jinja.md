# Ansible Guide: Jinja2 Templates, the `debug` Module, and Roles

A practical reference, using the Tomcat 10 installation as the running example.

---

## 1. Jinja2 Templates

### 1.1 What is it?

Ansible uses **Jinja2**, a Python templating engine, in two places:

1. **Inside playbooks/vars**: any value can contain `{{ variable }}`.
2. **In template files** (`*.j2`): config files rendered on the control node, then copied to the target by the `template` module.

### 1.2 The three Jinja delimiters

| Syntax | Purpose | Example |
|---|---|---|
| `{{ ... }}` | Output an expression/variable | `{{ tomcat_http_port }}` |
| `{% ... %}` | Logic: `if`, `for`, `set` | `{% if ssl_enabled %}` |
| `{# ... #}` | Comment (not rendered) | `{# managed by Ansible #}` |

> **YAML gotcha:** if a value *starts* with `{{`, you must quote it.
> `port: "{{ tomcat_http_port }}"` ✅  `port: {{ tomcat_http_port }}` ❌

### 1.3 `template` vs `copy`

| Module | Use when |
|---|---|
| `copy` | The file is static (same on every host). |
| `template` | The file needs variables, facts, loops or conditionals. |

```yaml
- name: Render systemd unit from template
  ansible.builtin.template:
    src: tomcat.service.j2        # looked up in the role's templates/ dir
    dest: /etc/systemd/system/tomcat.service
    owner: root
    group: root
    mode: "0644"
    validate: "systemd-analyze verify %s"   # optional: validate before replacing
  notify: Restart tomcat
```

Templates are **idempotent**: the file is only changed (and handlers only fire) if the rendered output differs from what is on the host.

### 1.4 Example template: `templates/tomcat.service.j2`

```jinja
{# {{ ansible_managed }} #}
[Unit]
Description=Apache Tomcat {{ tomcat_version }}
After=network.target

[Service]
Type=forking
User={{ tomcat_user }}
Group={{ tomcat_group }}

Environment="JAVA_HOME={{ java_home }}"
Environment="CATALINA_HOME={{ tomcat_home }}"
Environment="CATALINA_BASE={{ tomcat_home }}"
Environment="CATALINA_PID={{ tomcat_home }}/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms{{ tomcat_xms }} -Xmx{{ tomcat_xmx | default('1024M') }} -server"

ExecStart={{ tomcat_home }}/bin/startup.sh
ExecStop={{ tomcat_home }}/bin/shutdown.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 1.5 Variables, facts and magic variables

Anything Ansible knows can be used in a template:

| Kind | Example |
|---|---|
| Your variables | `{{ tomcat_version }}` |
| Facts (from `gather_facts`) | `{{ ansible_hostname }}`, `{{ ansible_memtotal_mb }}`, `{{ ansible_default_ipv4.address }}` |
| Magic variables | `{{ inventory_hostname }}`, `{{ groups['tomcat_servers'] }}`, `{{ hostvars['web1'].ansible_host }}` |

Example: size the heap to half of the server's RAM:

```jinja
CATALINA_OPTS=-Xmx{{ (ansible_memtotal_mb / 2) | int }}M
```

### 1.6 Conditionals (`if / elif / else`)

```jinja
{% if tomcat_ssl_enabled | default(false) %}
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           SSLEnabled="true" />
{% else %}
<Connector port="{{ tomcat_http_port }}" protocol="HTTP/1.1"
           connectionTimeout="20000" redirectPort="8443" />
{% endif %}
```

### 1.7 Loops (`for`)

Variable (in `defaults/main.yml`):

```yaml
tomcat_users:
  - { name: admin,  password: "s3cret",  roles: "manager-gui,admin-gui" }
  - { name: deploy, password: "d3ploy",  roles: "manager-script" }
```

Template `tomcat-users.xml.j2`:

```jinja
<tomcat-users>
{% for user in tomcat_users %}
  <user username="{{ user.name }}" password="{{ user.password }}" roles="{{ user.roles }}"/>
{% endfor %}
</tomcat-users>
```

Useful loop variables inside `{% for %}`: `loop.index` (1-based), `loop.first`, `loop.last`.

Looping over a host group (e.g. build a cluster member list):

```jinja
{% for host in groups['tomcat_servers'] %}
<Member className="org.apache.catalina.tribes.membership.StaticMember"
        host="{{ hostvars[host].ansible_default_ipv4.address }}" port="4000"/>
{% endfor %}
```

### 1.8 Filters (the `|` pipe)

| Filter | Example | Result |
|---|---|---|
| `default` | `{{ port \| default(8080) }}` | Fallback if undefined |
| `mandatory` | `{{ db_pass \| mandatory }}` | Fail if undefined |
| `upper` / `lower` | `{{ env \| upper }}` | `PROD` |
| `join` | `{{ ['a','b'] \| join(',') }}` | `a,b` |
| `int` / `bool` / `string` | `{{ "8080" \| int }}` | `8080` |
| `to_nice_json` / `to_nice_yaml` | `{{ mydict \| to_nice_json }}` | Pretty output |
| `regex_replace` | `{{ v \| regex_replace('\\.\\d+$','') }}` | Strip last part |
| `basename` / `dirname` | `{{ '/a/b/c.tgz' \| basename }}` | `c.tgz` |
| `length` | `{{ list \| length }}` | Count |
| `password_hash` | `{{ 'pw' \| password_hash('sha512') }}` | Hashed string |
| `b64encode` | `{{ 'text' \| b64encode }}` | Base64 |

### 1.9 Tests (the `is` keyword)

```yaml
when: tomcat_manager_allow is defined
when: ansible_facts['os_family'] is match("Debian")
when: result.rc is not failed
when: path_stat.stat.exists
```

### 1.10 Whitespace control

Block tags leave blank lines behind. Add `-` to trim:

```jinja
{%- if enabled %}
...
{%- endif %}
```

Or enable globally per file with the first line: `#jinja2: trim_blocks: True, lstrip_blocks: True`.

### 1.11 Useful built-in variable: `ansible_managed`

Put `{{ ansible_managed }}` in a comment at the top of config templates so people know not to hand-edit them. Its text can be customised in `ansible.cfg`.

### 1.12 Debugging a template

- Render without deploying: `ansible-playbook site.yml --check --diff` shows the diff of the rendered file.
- Print a variable first with the `debug` module (next section).

---

## 2. The `debug` Module

### 2.1 What is it?

`ansible.builtin.debug` prints messages or variable values during a run. It never changes the host (always `ok`, never `changed`) and is the main tool for troubleshooting.

### 2.2 Parameters

| Parameter | Meaning |
|---|---|
| `msg` | Text to print (supports Jinja). Default: `"Hello world!"` |
| `var` | Name of a variable to print. **Give the name only, without `{{ }}`**. |
| `verbosity` | Only print when run with at least that many `-v` flags (`0` default). |

`msg` and `var` are mutually exclusive.

### 2.3 Examples

**Print a message:**

```yaml
- name: Show what we are about to do
  ansible.builtin.debug:
    msg: "Installing Tomcat {{ tomcat_version }} on {{ inventory_hostname }}"
```

**Print a variable (note: no braces):**

```yaml
- name: Show Java home
  ansible.builtin.debug:
    var: java_home
```

**Inspect a registered result** (the most common use):

```yaml
- name: Check if Tomcat is already installed
  ansible.builtin.stat:
    path: /opt/tomcat/bin/catalina.sh
  register: tomcat_stat

- name: Show stat result
  ansible.builtin.debug:
    var: tomcat_stat

- name: Show just one field
  ansible.builtin.debug:
    msg: "Tomcat installed: {{ tomcat_stat.stat.exists }}"
```

**Print command output:**

```yaml
- name: Check Java version
  ansible.builtin.command: java -version
  register: java_out
  changed_when: false

- ansible.builtin.debug:
    msg: "{{ java_out.stderr_lines }}"    # java -version prints to stderr
```

**Print only with `-v`:**

```yaml
- ansible.builtin.debug:
    var: hostvars[inventory_hostname]
    verbosity: 2        # shown only with  ansible-playbook -vv
```

**Dump all facts about the host:**

```yaml
- ansible.builtin.debug:
    var: ansible_facts
```

**Debug inside a loop:**

```yaml
- ansible.builtin.debug:
    msg: "User {{ item.name }} gets roles {{ item.roles }}"
  loop: "{{ tomcat_users }}"
  loop_control:
    label: "{{ item.name }}"      # keeps output short and hides passwords
```

**Conditional debug:**

```yaml
- ansible.builtin.debug:
    msg: "Port {{ tomcat_http_port }} is non-standard"
  when: tomcat_http_port != 8080
```

### 2.4 Related tools

| Tool | Purpose |
|---|---|
| `-v`, `-vv`, `-vvv` | More output (module args, connection details). |
| `--check` | Dry run: show what *would* change. |
| `--diff` | Show line-by-line file differences. |
| `--step` | Confirm each task interactively. |
| `--start-at-task "name"` | Resume from a task. |
| `ansible.builtin.assert` | Fail early if a condition is false. |
| `ansible.builtin.fail` | Stop with a custom message. |
| `ansible-playbook --syntax-check` | Validate YAML/playbook syntax. |
| `ansible-inventory --list` | Show how inventory resolves. |
| `ansible -m setup host` | View facts for a host. |

### 2.5 Security tip

`debug` prints to the terminal and logs. Never debug passwords/secrets. Use `no_log: true` on tasks that handle secrets, and `loop_control.label` to hide loop items.

---

## 3. Ansible Roles

### 3.1 What is a role?

A **role** is a standard, self-contained folder layout that bundles tasks, variables, templates, files and handlers for one purpose (e.g. "install Tomcat"). Roles make automation:

- **Reusable**: same role in many playbooks/projects.
- **Organised**: no 500-line playbooks.
- **Shareable**: publish on Ansible Galaxy or Git.
- **Testable**: use Molecule to test a role independently.

Ansible automatically looks in the right sub-folder for each kind of content. You don't need to write paths like `src: ../templates/x.j2`.

### 3.2 Creating a role

```bash
ansible-galaxy role init tomcat        # creates the skeleton
# or into a roles/ directory:
ansible-galaxy role init roles/tomcat
```

### 3.3 Role directory structure

```text
roles/
└── tomcat/
    ├── defaults/
    │   └── main.yml        # Default variables (LOWEST precedence, easy to override)
    ├── vars/
    │   └── main.yml        # Role variables (HIGH precedence, internal constants)
    ├── tasks/
    │   └── main.yml        # The task list (entry point)
    ├── handlers/
    │   └── main.yml        # Handlers (restart, reload...)
    ├── templates/
    │   └── tomcat.service.j2   # Jinja2 templates (used by `template` module)
    ├── files/
    │   └── setenv.sh       # Static files (used by `copy` module)
    ├── meta/
    │   └── main.yml        # Role metadata and dependencies
    ├── library/            # (optional) custom modules
    ├── module_utils/       # (optional) shared module code
    ├── lookup_plugins/     # (optional) custom plugins
    ├── tests/
    │   ├── inventory
    │   └── test.yml        # Simple test playbook
    └── README.md           # Documentation
```

### 3.4 What each directory does

| Directory | Loaded automatically? | Purpose |
|---|---|---|
| `tasks/main.yml` | Yes | Tasks the role runs. Can `include_tasks` / `import_tasks` other files in the same folder. |
| `handlers/main.yml` | Yes | Handlers, triggered via `notify`. |
| `defaults/main.yml` | Yes | Settings users are expected to override. Lowest precedence. |
| `vars/main.yml` | Yes | Internal values that shouldn't normally be overridden. High precedence. |
| `templates/` | Referenced by name | `.j2` files for `ansible.builtin.template`. |
| `files/` | Referenced by name | Static files for `copy`, `script`, etc. |
| `meta/main.yml` | Yes | Author info, supported platforms, **role dependencies**. |
| `library/` etc. | Yes | Custom modules/plugins available only to this role. |

Only `main.yml` is loaded automatically. Extra files (e.g. `tasks/install.yml`) must be included explicitly.

### 3.5 Project layout around roles

```text
tomcat-project/
├── ansible.cfg
├── inventory/
│   ├── production.ini
│   ├── staging.ini
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── tomcat_servers.yml     # overrides role defaults per group
│   └── host_vars/
│       └── web1.example.com.yml   # overrides per host
├── site.yml                       # top-level playbook
├── playbooks/
│   └── tomcat.yml
├── requirements.yml               # roles/collections from Galaxy
└── roles/
    ├── common/
    ├── java/
    └── tomcat/
```

### 3.6 Using a role in a playbook

**Classic `roles:` keyword** (runs before `tasks:`):

```yaml
# site.yml
- name: Deploy Tomcat
  hosts: tomcat_servers
  become: true
  roles:
    - common
    - java
    - role: tomcat
      vars:
        tomcat_version: "10.1.34"
        tomcat_http_port: 8080
```

**Dynamic/static include** (run a role in the middle of tasks):

```yaml
  tasks:
    - name: Pre-check
      ansible.builtin.debug:
        msg: "Starting"

    - name: Apply tomcat role
      ansible.builtin.include_role:      # dynamic: evaluated at runtime, supports loops/when
        name: tomcat
      vars:
        tomcat_version: "10.1.34"

    - ansible.builtin.import_role:       # static: parsed at playbook load
        name: tomcat
        tasks_from: install               # run only tasks/install.yml
```

Execution order in a play: `pre_tasks` → handlers flush → `roles` → `tasks` → handlers flush → `post_tasks`.

### 3.7 Variable precedence (simplified, low → high)

1. `roles/x/defaults/main.yml`
2. Inventory vars / `group_vars/all` → `group_vars/<group>` → `host_vars/<host>`
3. Play `vars:`
4. `roles/x/vars/main.yml`
5. Task/block vars, role `vars:` passed in the playbook
6. `set_fact` / `register`
7. Extra vars: `-e "tomcat_version=10.1.40"` (always wins)

**Rule of thumb:** put every tunable in `defaults/`, keep `vars/` for constants. Prefix variable names with the role name (`tomcat_port`, not `port`) to avoid collisions.

### 3.8 Role dependencies (`meta/main.yml`)

```yaml
---
galaxy_info:
  author: your-name
  description: Install and configure Apache Tomcat 10
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]

dependencies:
  - role: java
    vars:
      java_package: default-jdk
```

Dependencies run **before** the role that lists them.

---

## 4. Complete Example: Tomcat as a Role

### 4.1 Layout

```text
tomcat-project/
├── inventory.ini
├── site.yml
└── roles/
    └── tomcat/
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── meta/main.yml
        ├── tasks/
        │   ├── main.yml
        │   ├── install.yml
        │   └── configure.yml
        └── templates/
            ├── tomcat.service.j2
            └── tomcat-users.xml.j2
```

### 4.2 `inventory.ini`

```ini
[tomcat_servers]
web1 ansible_host=203.0.113.10 ansible_user=ubuntu
web2 ansible_host=203.0.113.11 ansible_user=ubuntu
```

### 4.3 `site.yml`

```yaml
---
- name: Install Tomcat 10
  hosts: tomcat_servers
  become: true
  roles:
    - tomcat
```

### 4.4 `roles/tomcat/defaults/main.yml`

```yaml
---
tomcat_version: "10.1.34"
tomcat_user: tomcat
tomcat_group: tomcat
tomcat_home: /opt/tomcat
tomcat_http_port: 8080
java_home: /usr/lib/jvm/java-1.11.0-openjdk-amd64
tomcat_xms: 512M
tomcat_xmx: 1024M
tomcat_download_url: "https://archive.apache.org/dist/tomcat/tomcat-10/v{{ tomcat_version }}/bin/apache-tomcat-{{ tomcat_version }}.tar.gz"

tomcat_users:
  - name: admin
    password: "{{ vault_tomcat_admin_password | default('ChangeMe!') }}"
    roles: "manager-gui,admin-gui"
```

### 4.5 `roles/tomcat/tasks/main.yml`

```yaml
---
- name: Show what will happen
  ansible.builtin.debug:
    msg: "Installing Tomcat {{ tomcat_version }} on {{ inventory_hostname }}"

- name: Install prerequisites and Tomcat
  ansible.builtin.import_tasks: install.yml

- name: Configure Tomcat
  ansible.builtin.import_tasks: configure.yml

- name: Enable and start Tomcat
  ansible.builtin.systemd:
    name: tomcat
    enabled: true
    state: started
    daemon_reload: true
```

### 4.6 `roles/tomcat/tasks/install.yml`

```yaml
---
- name: Install Java
  ansible.builtin.apt:
    name: default-jdk
    state: present
    update_cache: true
    cache_valid_time: 3600

- name: Create tomcat group
  ansible.builtin.group:
    name: "{{ tomcat_group }}"
    system: true

- name: Create tomcat user
  ansible.builtin.user:
    name: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    home: "{{ tomcat_home }}"
    shell: /bin/false
    system: true
    create_home: false

- name: Create install directory
  ansible.builtin.file:
    path: "{{ tomcat_home }}"
    state: directory
    owner: root
    group: "{{ tomcat_group }}"
    mode: "0755"

- name: Download and extract Tomcat
  ansible.builtin.unarchive:
    src: "{{ tomcat_download_url }}"
    dest: "{{ tomcat_home }}"
    remote_src: true
    extra_opts: ["--strip-components=1"]
    creates: "{{ tomcat_home }}/bin/catalina.sh"
  notify: Restart tomcat

- name: Set ownership of runtime directories
  ansible.builtin.file:
    path: "{{ tomcat_home }}/{{ item }}"
    state: directory
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    recurse: true
  loop: [webapps, work, temp, logs]
```

### 4.7 `roles/tomcat/tasks/configure.yml`

```yaml
---
- name: Install systemd unit (from Jinja2 template)
  ansible.builtin.template:
    src: tomcat.service.j2
    dest: /etc/systemd/system/tomcat.service
    owner: root
    group: root
    mode: "0644"
  notify:
    - Reload systemd
    - Restart tomcat

- name: Configure Tomcat users (from Jinja2 template)
  ansible.builtin.template:
    src: tomcat-users.xml.j2
    dest: "{{ tomcat_home }}/conf/tomcat-users.xml"
    owner: root
    group: "{{ tomcat_group }}"
    mode: "0640"
  no_log: true
  notify: Restart tomcat
```

### 4.8 `roles/tomcat/templates/tomcat-users.xml.j2`

```jinja
<?xml version="1.0" encoding="UTF-8"?>
{# {{ ansible_managed }} #}
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
{% for user in tomcat_users %}
  <user username="{{ user.name }}" password="{{ user.password }}" roles="{{ user.roles }}"/>
{% endfor %}
</tomcat-users>
```

### 4.9 `roles/tomcat/handlers/main.yml`

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

### 4.10 Run it

```bash
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml --check --diff     # dry run
ansible-playbook -i inventory.ini site.yml                    # real run
ansible-playbook -i inventory.ini site.yml -e tomcat_version=10.1.40
```

---

## 5. Quick Cheat Sheet

| Goal | Tool |
|---|---|
| Static file to host | `copy` (file in role's `files/`) |
| Dynamic config file | `template` (file in role's `templates/`) |
| Print a value while debugging | `debug: var=name` or `debug: msg="{{ name }}"` |
| Make a default overridable | Put it in `defaults/main.yml` |
| Run something only on change | `notify` + `handlers/main.yml` |
| Run another role first | `meta/main.yml` → `dependencies` |
| Create role skeleton | `ansible-galaxy role init <name>` |
| Download shared roles | `ansible-galaxy install -r requirements.yml` |
| Encrypt secrets | `ansible-vault encrypt_string` / `ansible-vault create` |
| Lint your code | `ansible-lint` |

### Best practices

1. One role = one responsibility (java, tomcat, nginx).
2. Prefix role variables (`tomcat_*`).
3. Use fully qualified module names (`ansible.builtin.copy`).
4. Keep secrets in Ansible Vault, never in plain-text vars.
5. Name every task. The names are your run log.
6. Prefer modules over `shell`/`command`; if you must use them, set `changed_when`.
7. Always test with `--check --diff` before production.
8. Document variables in the role's `README.md`.
