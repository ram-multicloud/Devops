# Understanding and Creating Ansible Roles

## What is an Ansible Role?
An Ansible Role is a concept that allows you to break down a complex playbook into multiple files and directories based on their function (tasks, variables, handlers, templates, etc.). 

Instead of writing a massive single playbook file, roles allow you to:
1. **Reuse Code:** You can easily share and reuse roles across different projects (or even share them on Ansible Galaxy).
2. **Organize Structure:** They enforce a standard directory structure, making it much easier for other engineers to read and maintain your code.
3. **Simplify Playbooks:** Your main playbook becomes extremely clean, often just mapping hosts to roles.

## How to Create an Ansible Role
The easiest way to generate the standard directory structure for an Ansible role is by using the `ansible-galaxy` command-line tool.

To create a role named `tomcat`, you would run:
```bash
ansible-galaxy init tomcat
```
This command generates a directory named `tomcat` with several subdirectories (`tasks`, `handlers`, `defaults`, `vars`, `templates`, etc.), each containing a `main.yml` file.

---

## Converting Your Playbook to a Tomcat Role

Based on the playbook you provided, here is how you distribute the different parts into the standard role structure. 

### Directory Structure
Your project directory should look like this:

```text
project-directory/
├── deploy-tomcat.yml       # Your main playbook that calls the role
└── roles/
    └── tomcat/
        ├── defaults/
        │   └── main.yml    # Your variables go here
        ├── tasks/
        │   └── main.yml    # Your tasks go here
        └── handlers/
            └── main.yml    # Your handlers go here
```

### 1. Variables (`roles/tomcat/defaults/main.yml`)
Move everything under the `vars:` section into the `defaults/main.yml` file. Using `defaults` is best practice because it allows you to easily override these variables later if needed.

```yaml
---
# roles/tomcat/defaults/main.yml

tomcat_group: tomcat
tomcat_app_version: tomcat-10
java_package: openjdk-25-jdk
tomcat_version: 10.1.61
tomcat_user: tomcat
home: /opt/tomcat
url: "https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.61/bin/apache-tomcat-10.1.61.tar.gz"
```

### 2. Tasks (`roles/tomcat/tasks/main.yml`)
Extract the list of tasks from your playbook and put them here. Remove the playbook-level directives like `hosts`, `become`, and `vars`.

```yaml
---
# roles/tomcat/tasks/main.yml

- name: java installation
  ansible.builtin.apt:
    name: "{{ java_package }}"
    state: present
    update_cache: yes

- name: create tomcat group
  ansible.builtin.group:
    name: "{{ tomcat_group }}"
    system: yes
    state: present

- name: create tomcat user
  ansible.builtin.user:
    name: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"
    home: "{{ home }}"
    shell: /bin/false
    system: yes
    state: present

- name: download tomcat
  ansible.builtin.get_url:
    url: "{{ url }}"
    dest: "/tmp/apache-tomcat-{{ tomcat_version }}.tar.gz"

- name: extract tomcat
  ansible.builtin.unarchive:
    src: "/tmp/apache-tomcat-{{ tomcat_version }}.tar.gz"
    dest: /opt/tomcat
    remote_src: yes

- name: change ownership of tomcat directory
  ansible.builtin.file:
    path: /opt/tomcat
    state: directory
    recurse: yes
    owner: "{{ tomcat_user }}"
    group: "{{ tomcat_group }}"

- name: provide execute permission to tomcat scripts
  ansible.builtin.file:
    path: /opt/tomcat/bin
    state: directory
    recurse: yes
    mode: u+x

- name: update java alternatives
  ansible.builtin.shell:
    cmd: update-java-alternatives -l

- name: create tomcat service file
  ansible.builtin.copy:
    dest: /etc/systemd/system/tomcat.service
    content: |
      [Unit]
      Description=Tomcat
      After=network.target

      [Service]
      Type=forking

      User=tomcat
      Group=tomcat

      Environment="JAVA_HOME=/usr/lib/jvm/java-1.25.0-openjdk-amd64"
      Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom"
      Environment="CATALINA_BASE=/opt/tomcat"
      Environment="CATALINA_HOME=/opt/tomcat"
      Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
      Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

      ExecStart=/opt/tomcat/bin/startup.sh
      ExecStop=/opt/tomcat/bin/shutdown.sh

      RestartSec=10
      Restart=always

      [Install]
      WantedBy=multi-user.target
  notify:
    - reload systemd daemon
    - start tomcat service
```

### 3. Handlers (`roles/tomcat/handlers/main.yml`)
Move your handlers list into the handlers file.

```yaml
---
# roles/tomcat/handlers/main.yml

- name: reload systemd daemon
  ansible.builtin.systemd:
    daemon_reload: yes

- name: start tomcat service
  ansible.builtin.systemd:
    name: tomcat
    state: started
    enabled: yes
```

### 4. The Main Playbook (`deploy-tomcat.yml`)
Finally, you replace your massive original file with a simple playbook that calls your newly created role. Notice how clean it is now!

```yaml
---
# deploy-tomcat.yml (in your main project folder)

- name: Deploy Tomcat Server
  hosts: all
  become: true
  roles:
    - tomcat
```

### Running Your Role
You execute this exactly as you would execute your original playbook:
```bash
ansible-playbook deploy-tomcat.yml -i your_inventory_file
```
