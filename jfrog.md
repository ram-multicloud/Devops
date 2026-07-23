# Installing JFrog Artifactory OSS on Ubuntu

## 1. Prerequisites

```bash
sudo apt-get update
sudo apt-get install -y openjdk-17-jdk wget tar
```

Check disk space before continuing — Artifactory needs several GB free:

```bash
df -h /
```

If space is low:

```bash
sudo apt-get clean
sudo journalctl --vacuum-size=50M
```

## 2. Download and extract

```bash
export VERSION="7.146.7"
export JFROG_HOME="/opt/jfrog"

sudo mkdir -p "$JFROG_HOME"
cd /tmp
wget -N "https://releases.jfrog.io/artifactory/bintray-artifactory/org/artifactory/oss/jfrog-artifactory-oss/${VERSION}/jfrog-artifactory-oss-${VERSION}-linux.tar.gz"
sudo tar -xvzf "jfrog-artifactory-oss-${VERSION}-linux.tar.gz" -C "$JFROG_HOME"
```

Rename the extracted folder to `artifactory` (check the actual extracted name first with `ls $JFROG_HOME`):

```bash
sudo mv "$JFROG_HOME/artifactory-oss-${VERSION}" "$JFROG_HOME/artifactory"
```

## 3. Install as a systemd service

```bash
cd "$JFROG_HOME/artifactory/app/bin"
sudo ./installService.sh
```

## 4. Configure the database

Artifactory 7.x refuses to start on the embedded Derby DB unless explicitly allowed.

**Option A — allow embedded DB (quick, for test/eval only):**

```bash
sudo touch /opt/jfrog/artifactory/var/etc/system.yaml
sudo tee -a /opt/jfrog/artifactory/var/etc/system.yaml > /dev/null << 'EOF'
shared:
  database:
    allowNonPostgresql: true
EOF
sudo chown artifactory:artifactory /opt/jfrog/artifactory/var/etc/system.yaml
```

**Option B — PostgreSQL (recommended for real use):**

```bash
sudo apt-get install -y postgresql
sudo -u postgres psql -c "CREATE USER artifactory WITH PASSWORD 'password123';"
sudo -u postgres psql -c "CREATE DATABASE artifactory OWNER artifactory;"

sudo tee /opt/jfrog/artifactory/var/etc/system.yaml > /dev/null << 'EOF'
shared:
  database:
    type: postgresql
    driver: org.postgresql.Driver
    url: jdbc:postgresql://localhost:5432/artifactory
    username: artifactory
    password: password123
EOF
sudo chown artifactory:artifactory /opt/jfrog/artifactory/var/etc/system.yaml
```

## 5. Start the service

```bash
sudo systemctl daemon-reload
sudo systemctl start artifactory.service
sudo systemctl enable artifactory.service
sudo systemctl status artifactory.service
```

## 6. Verify and access

```bash
sudo ss -tlnp | grep 8082
sudo tail -f /opt/jfrog/artifactory/var/log/console.log
curl -I http://localhost:8082/
```

Open `http://<server-ip>:8082/` in a browser. Default login: `admin` / `password`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `mv: cannot overwrite ... Directory not empty` | Previous partial install left `/opt/jfrog/artifactory` in place | `sudo rm -rf /opt/jfrog/artifactory` before re-extracting |
| `No space left on device` during `apt-get` | Disk full | `df -h /`, then `apt-get clean` / grow the EBS volume |
| `gpgv` signature errors during `apt-get update` | Usually a symptom of the disk-space issue above, not gnupg itself | Free space first, then retry |
| Service active but UI unreachable | Not fully started yet, firewall/security group blocking 8082, or DB not initialized | Check `console.log`; check `ufw status`; check AWS security group inbound rules for port 8082 |
| `Cannot start the application with a database other than PostgreSQL` | Artifactory 7.146.x blocks the embedded Derby DB by default | Set `allowNonPostgresql: true` in `system.yaml`, or configure PostgreSQL |
| `Failed resolving MasterKey key` | Downstream failure caused by the DB init failure above | Resolves once the DB issue is fixed |

## Notes

- Logs: `/opt/jfrog/artifactory/var/log/`
- Config: `/opt/jfrog/artifactory/var/etc/system.yaml`
- Default UI/API port: `8082`
