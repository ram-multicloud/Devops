# Install nopCommerce on Ubuntu with Ansible

Automates the official guide:
<https://docs.nopcommerce.com/en/installation-and-upgrading/installing-nopcommerce/installing-on-linux.html>

## What the playbook installs

| Component | Purpose |
|---|---|
| ASP.NET Core 8 runtime (+ `libgdiplus`) | Runs nopCommerce |
| MySQL 8 | Database (dedicated DB and user are created) |
| nginx | Reverse proxy from port 80 to the app on port 5000 |
| nopCommerce 4.70.4 | The store, unpacked to `/var/www/nopCommerce` |
| systemd service `nopCommerce` | Keeps the app running and restarts it on crash |

## Project layout

```
nopcommerce-ansible/
├── inventory.ini
├── requirements.yml
└── nopcommerce.yml
```

## Prerequisites

- Control machine with Ansible installed
- Target server running Ubuntu (SSH access and sudo rights)
- Internet access on the target (to download packages and the nopCommerce release)

## Files

### `inventory.ini`

```ini
[nopcommerce]
nop-server ansible_host=203.0.113.10 ansible_user=ubuntu
```

### `requirements.yml`

```yaml
---
collections:
  - name: community.mysql
```

### `nopcommerce.yml`

```yaml
---
# Install nopCommerce on Ubuntu (nginx reverse proxy + MySQL 8 + ASP.NET Core 8)
# Based on: https://docs.nopcommerce.com/en/installation-and-upgrading/installing-nopcommerce/installing-on-linux.html
#
# Usage:
#   ansible-galaxy collection install -r requirements.yml
#   ansible-playbook -i inventory.ini nopcommerce.yml --ask-become-pass \
#       -e db_password='YourStrongPassword' -e server_name=shop.example.com
#
# Tip: keep db_password in an ansible-vault file instead of the command line.

- name: Install nopCommerce
  hosts: nopcommerce
  become: true

  vars:
    # --- nopCommerce ---
    nop_version: "4.70.4"
    nop_zip: "nopCommerce_{{ nop_version }}_NoSource_linux_x64.zip"
    nop_url: "https://github.com/nopSolutions/nopCommerce/releases/download/release-{{ nop_version }}/{{ nop_zip }}"
    nop_dir: /var/www/nopCommerce
    nop_user: www-data
    nop_group: www-data
    nop_port: 5000

    # --- Runtime ---
    dotnet_package: aspnetcore-runtime-8.0
    # The guide registers the Microsoft feed (Ubuntu 20.04). Ubuntu 22.04+ ships
    # .NET 8 in its own repositories, so the feed is only needed on older releases.
    register_microsoft_feed: "{{ ansible_distribution_version is version('22.04', '<') }}"

    # --- Web server ---
    server_name: nopcommerce.example.com

    # --- Database (dedicated app user instead of using MySQL root) ---
    db_name: nopcommerce
    db_user: nopuser
    db_password: "ChangeMe_Str0ng!"   # CHANGE THIS or override with -e / ansible-vault

  pre_tasks:
    - name: Make sure the target is Ubuntu
      ansible.builtin.assert:
        that: ansible_distribution == "Ubuntu"
        fail_msg: "This playbook supports Ubuntu only."

  tasks:
    # ------------------------------------------------------------------
    # 1. Base packages
    # ------------------------------------------------------------------
    - name: Install base packages
      ansible.builtin.apt:
        name:
          - apt-transport-https
          - ca-certificates
          - unzip
          - python3-pymysql      # required by community.mysql modules
        state: present
        update_cache: true

    # ------------------------------------------------------------------
    # 2. .NET runtime
    # ------------------------------------------------------------------
    - name: Download Microsoft package feed registration
      ansible.builtin.get_url:
        url: "https://packages.microsoft.com/config/ubuntu/{{ ansible_distribution_version }}/packages-microsoft-prod.deb"
        dest: /tmp/packages-microsoft-prod.deb
        mode: "0644"
      when: register_microsoft_feed | bool

    - name: Register Microsoft package feed
      ansible.builtin.apt:
        deb: /tmp/packages-microsoft-prod.deb
      when: register_microsoft_feed | bool

    - name: Install ASP.NET Core runtime
      ansible.builtin.apt:
        name: "{{ dotnet_package }}"
        state: present
        update_cache: true

    - name: Install libgdiplus (fixes 'Gdip' image errors in the RichText box)
      ansible.builtin.apt:
        name: libgdiplus
        state: present

    # ------------------------------------------------------------------
    # 3. MySQL
    # ------------------------------------------------------------------
    - name: Install MySQL server
      ansible.builtin.apt:
        name: mysql-server
        state: present

    - name: Ensure MySQL is running and enabled
      ansible.builtin.service:
        name: mysql
        state: started
        enabled: true

    - name: Create nopCommerce database
      community.mysql.mysql_db:
        name: "{{ db_name }}"
        encoding: utf8mb4
        collation: utf8mb4_unicode_ci
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Create nopCommerce database user
      community.mysql.mysql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        host: localhost
        priv: "{{ db_name }}.*:ALL"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
      no_log: true

    # ------------------------------------------------------------------
    # 4. nginx reverse proxy
    # ------------------------------------------------------------------
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Write nginx site configuration
      ansible.builtin.copy:
        dest: /etc/nginx/sites-available/nopcommerce
        owner: root
        group: root
        mode: "0644"
        content: |
          server {
              listen 80 default_server;
              listen [::]:80 default_server;
              server_name {{ server_name }};

              location / {
                  proxy_pass http://localhost:{{ nop_port }};
                  proxy_http_version 1.1;
                  proxy_set_header Upgrade $http_upgrade;
                  proxy_set_header Connection keep-alive;
                  proxy_set_header Host $host;
                  proxy_cache_bypass $http_upgrade;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
              }
          }
      notify: Restart nginx

    - name: Enable nopCommerce site
      ansible.builtin.file:
        src: /etc/nginx/sites-available/nopcommerce
        dest: /etc/nginx/sites-enabled/nopcommerce
        state: link
      notify: Restart nginx

    - name: Disable the default nginx site
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Restart nginx

    - name: Validate nginx configuration
      ansible.builtin.command: nginx -t
      changed_when: false

    - name: Ensure nginx is running and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    # ------------------------------------------------------------------
    # 5. nopCommerce application
    # ------------------------------------------------------------------
    - name: Create application directory
      ansible.builtin.file:
        path: "{{ nop_dir }}"
        state: directory
        owner: "{{ nop_user }}"
        group: "{{ nop_group }}"
        mode: "0755"

    - name: Download nopCommerce release
      ansible.builtin.get_url:
        url: "{{ nop_url }}"
        dest: "/tmp/{{ nop_zip }}"
        mode: "0644"

    - name: Unpack nopCommerce
      ansible.builtin.unarchive:
        src: "/tmp/{{ nop_zip }}"
        dest: "{{ nop_dir }}"
        remote_src: true
        owner: "{{ nop_user }}"
        group: "{{ nop_group }}"
        creates: "{{ nop_dir }}/Nop.Web.dll"

    - name: Create bin and logs directories
      ansible.builtin.file:
        path: "{{ nop_dir }}/{{ item }}"
        state: directory
        owner: "{{ nop_user }}"
        group: "{{ nop_group }}"
        mode: "0755"
      loop:
        - bin
        - logs

    - name: Set ownership on the application tree
      ansible.builtin.file:
        path: "{{ nop_dir }}"
        state: directory
        recurse: true
        owner: "{{ nop_user }}"
        group: "{{ nop_group }}"

    # ------------------------------------------------------------------
    # 6. systemd service
    # ------------------------------------------------------------------
    - name: Create nopCommerce systemd unit
      ansible.builtin.copy:
        dest: /etc/systemd/system/nopCommerce.service
        owner: root
        group: root
        mode: "0644"
        content: |
          [Unit]
          Description=nopCommerce app running on Ubuntu

          [Service]
          WorkingDirectory={{ nop_dir }}
          ExecStart=/usr/bin/dotnet {{ nop_dir }}/Nop.Web.dll
          Restart=always
          # Restart service after 10 seconds if the dotnet service crashes:
          RestartSec=10
          KillSignal=SIGINT
          SyslogIdentifier=nopCommerce
          User={{ nop_user }}
          Environment=ASPNETCORE_ENVIRONMENT=Production
          Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

          [Install]
          WantedBy=multi-user.target
      notify:
        - Reload systemd
        - Restart nopCommerce

    - name: Apply pending handlers before starting services
      ansible.builtin.meta: flush_handlers

    - name: Enable and start nopCommerce
      ansible.builtin.systemd:
        name: nopCommerce
        enabled: true
        state: started

    # ------------------------------------------------------------------
    # 7. Smoke tests
    # ------------------------------------------------------------------
    - name: Wait for nopCommerce to listen on its port
      ansible.builtin.wait_for:
        host: 127.0.0.1
        port: "{{ nop_port }}"
        timeout: 90

    - name: Check the site through nginx (fresh installs redirect to /install)
      ansible.builtin.uri:
        url: http://127.0.0.1/
        follow_redirects: none
        status_code: [200, 301, 302]

    - name: Next steps
      ansible.builtin.debug:
        msg:
          - "nopCommerce is running. Open http://{{ server_name }}/ (or the server IP) to run the install wizard."
          - "Database server: localhost | name: {{ db_name }} | user: {{ db_user }}"
          - "In the wizard use the existing database (do not tick 'Create database if it doesn't exist')."
          - "For SSL, set UseProxy to true in {{ nop_dir }}/appsettings.json."

  handlers:
    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: true

    - name: Restart nopCommerce
      ansible.builtin.systemd:
        name: nopCommerce
        state: restarted

    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

## How to run

```bash
# 1. Install the required collection
ansible-galaxy collection install -r requirements.yml

# 2. Run the playbook
ansible-playbook -i inventory.ini nopcommerce.yml --ask-become-pass \
  -e db_password='YourStrongPassword' \
  -e server_name=shop.example.com
```

Tip: store `db_password` in an ansible-vault file instead of typing it on the command line.

## Variables you can change

| Variable | Default | Meaning |
|---|---|---|
| `nop_version` | `4.70.4` | nopCommerce release to install |
| `nop_dir` | `/var/www/nopCommerce` | Install location |
| `nop_port` | `5000` | Port the app listens on (nginx proxies to it) |
| `server_name` | `nopcommerce.example.com` | nginx server name |
| `db_name` | `nopcommerce` | MySQL database name |
| `db_user` | `nopuser` | MySQL application user |
| `db_password` | `ChangeMe_Str0ng!` | MySQL user password (**change it**) |
| `dotnet_package` | `aspnetcore-runtime-8.0` | .NET runtime package |

## What each stage does

1. **Base packages:** `unzip`, `ca-certificates`, and `python3-pymysql` (needed by the MySQL modules).
2. **.NET runtime:** registers the Microsoft feed only on Ubuntu older than 22.04, then installs `aspnetcore-runtime-8.0` and `libgdiplus`.
3. **MySQL:** installs the server, creates the database and a dedicated user.
4. **nginx:** writes a reverse-proxy site, enables it, disables the default site, and validates with `nginx -t`.
5. **nopCommerce:** downloads and unpacks the release, creates `bin` and `logs`, sets `www-data` ownership.
6. **systemd:** creates, enables, and starts the `nopCommerce` service.
7. **Smoke tests:** waits for port 5000 and checks that nginx responds.

## After the playbook finishes

1. Open `http://<server-ip-or-name>/` in a browser. A fresh install redirects to the install wizard.
2. In the wizard, enter the database details:
   - Server: `localhost`
   - Database: `nopcommerce`
   - User and password: the values of `db_user` and `db_password`
   - Use the **existing database** (do not tick "create database if it doesn't exist").
3. Finish the wizard and log in as the admin you created.

## Differences from the official guide

| Official guide | This playbook | Why |
|---|---|---|
| Registers the Microsoft feed (Ubuntu 20.04) | Only on Ubuntu older than 22.04 | 22.04+ ships .NET 8 in its own repositories |
| Runs `mysql_secure_installation` and sets a root password | Creates a dedicated DB user and leaves root alone | That script is interactive and not repeatable |
| Overwrites `sites-available/default` | Own site file; default site disabled | Cleaner and easier to undo |

## Troubleshooting

| Problem | Fix |
|---|---|
| `The type initializer for 'Gdip' threw an exception` | `libgdiplus` is installed by the playbook; re-run it if missing |
| Site not loading | `sudo systemctl status nopCommerce` and `sudo journalctl -u nopCommerce -e` |
| nginx error | `sudo nginx -t` and check `/var/log/nginx/error.log` |
| Using SSL | Set `UseProxy` to `true` in `/var/www/nopCommerce/appsettings.json` |
| Upgrading versions | The playbook skips unpacking if `Nop.Web.dll` exists; back up and follow the official upgrade guide |

## Security notes

- Change `db_password` and keep it in ansible-vault.
- Add HTTPS (for example with Let's Encrypt) before taking real orders.
- Restrict the firewall to ports 22, 80, and 443.
