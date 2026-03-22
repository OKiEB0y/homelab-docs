# Wazuh 4.11.2 All-in-One Install (Manager + Indexer + Dashboard)
**Date:** 2026-03-20
**Category:** services
**System:** Ubuntu 24.04.4 LTS ([SIEM-HOST] — [SIEM-HOST-IP])
**Risk Level:** Medium

---

## Overview

Installs the complete Wazuh 4.11.2 SIEM stack (indexer, manager, dashboard) on a single Ubuntu 24.04
server using the official Wazuh installation assistant. No agents are configured — this establishes
the server-side infrastructure ready for agent enrollment.

---

## Prerequisites

- Ubuntu 24.04.4 LTS
- 7.6 GB RAM (4.9 GB available + 4 GB swap — workable, tight)
- 400+ GB free disk
- Internet access for package downloads
- sudo privileges
- SSH access: `ssh -o StrictHostKeyChecking=no [admin-user]@[SIEM-HOST-IP]`

**Hardware reality check:**
Wazuh recommends 16 GB RAM for full stack. With 7.6 GB total and 2.7 GB available post-install,
the server runs but headroom is tight. Monitor memory over time. If swap usage climbs, consider
disabling the dashboard and running a standalone indexer + manager only, with a remote dashboard
pointed at the API.

---

## Pre-Change State / Backup

```
RAM before install:  4.9 GB available  (2.7 GB used)
RAM after install:   2.7 GB available  (4.9 GB used)
Swap before:         27 MB used / 4 GB
Swap after:          66 MB used / 4 GB
Disk (/):            400 GB free / 468 GB total
```

No pre-existing Wazuh installation — clean install, no backup needed.

---

## Step-by-Step Instructions

### 1. Check resources

```bash
ssh -o StrictHostKeyChecking=no [admin-user]@[SIEM-HOST-IP]
free -h && df -h /
```

### 2. Download installer and config template

```bash
cd ~
curl -sO https://packages.wazuh.com/4.11/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.11/config.yml
```

### 3. Edit config.yml for single-node deployment

Replace all placeholder IPs with the server IP ([SIEM-HOST-IP]):

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "[SIEM-HOST-IP]"
  server:
    - name: wazuh-1
      ip: "[SIEM-HOST-IP]"
  dashboard:
    - name: dashboard
      ip: "[SIEM-HOST-IP]"
```

### 4. Generate certificates and config files

```bash
echo '[REDACTED]' | sudo -S bash wazuh-install.sh --generate-config-files
```

Output: Creates `wazuh-install-files.tar` containing cluster key, TLS certificates, and passwords.

### 5. Install Wazuh Indexer (OpenSearch)

```bash
echo '[REDACTED]' | sudo -S bash wazuh-install.sh --wazuh-indexer node-1
```

Installs OpenSearch-based indexer, configures security, starts `wazuh-indexer` service.

### 6. Start the indexer cluster

```bash
echo '[REDACTED]' | sudo -S bash wazuh-install.sh --start-cluster
```

Initializes security configuration, updates internal users, starts the cluster.

### 7. Install Wazuh Server (manager + Filebeat)

```bash
echo '[REDACTED]' | sudo -S bash wazuh-install.sh --wazuh-server wazuh-1
```

Installs `wazuh-manager`, configures vulnerability detection, installs and starts Filebeat to
ship manager logs to the indexer.

### 8. Install Wazuh Dashboard

```bash
echo '[REDACTED]' | sudo -S bash wazuh-install.sh --wazuh-dashboard dashboard
```

Installs OpenSearch Dashboards + Wazuh plugin, binds to port 443 with TLS.

---

## Verification

### Service status check

```bash
for svc in wazuh-indexer wazuh-manager wazuh-dashboard filebeat; do
  echo "--- $svc ---"
  systemctl status $svc --no-pager | grep -E "Active:|Main PID"
done
```

Expected: all four services `active (running)` and set to enabled.

### Port bindings (confirmed post-install)

| Port  | Service                     | Purpose                              |
|-------|-----------------------------|--------------------------------------|
| 443   | wazuh-dashboard (node)      | Web UI (HTTPS)                       |
| 1514  | wazuh-remoted               | Agent event ingestion                |
| 1515  | wazuh-authd                 | Agent auto-enrollment                |
| 55000 | wazuh-manager API (python3) | REST API                             |
| 9200  | wazuh-indexer (java)        | OpenSearch REST (internal)           |

---

## Dashboard Access

**URL:** `https://[SIEM-HOST-IP]:443`
**Username:** `admin`
**Password:** `[REDACTED]`

Note: The browser will show a TLS certificate warning (self-signed cert). Accept the exception
to proceed. The warning is normal for a default install — replace with a trusted cert later if
exposing externally.

The first login may show a 401 error briefly while the indexer completes its warm-up. Wait 60
seconds and refresh if this occurs.

**Password storage:** Passwords are also stored at:
```bash
sudo tar -axf ~/wazuh-install-files.tar wazuh-passwords.txt -O
```

---

## Rollback / Recovery

### Full removal

```bash
# Stop all services
sudo systemctl stop wazuh-dashboard wazuh-manager wazuh-indexer filebeat

# Remove packages
sudo apt remove --purge -y wazuh-manager wazuh-indexer wazuh-dashboard filebeat

# Remove data directories
sudo rm -rf /var/ossec /etc/wazuh-* /var/lib/wazuh-* /var/log/wazuh-*

# Remove apt repo
sudo rm /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

### Restart stuck services

```bash
sudo systemctl restart wazuh-indexer
# Wait 30s for indexer to be ready, then:
sudo systemctl restart filebeat
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-dashboard
```

### Recover passwords if lost

```bash
sudo tar -axf ~/wazuh-install-files.tar wazuh-passwords.txt -O
```

---

## Adding Agents Later

### On the Wazuh manager server, generate an enrollment key

The manager auto-enrolls agents via port 1515 (`wazuh-authd`). No manual key needed — just
point agents at the manager IP.

### Linux agent install (on the agent machine)

```bash
# Add repo
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list

# Install agent, pointing to manager
sudo apt update
WAZUH_MANAGER="[SIEM-HOST-IP]" sudo apt install -y wazuh-agent

# Start
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

### Windows agent install

Run the official MSI from https://packages.wazuh.com/4.x/windows/ with:
```
WAZUH_MANAGER=[SIEM-HOST-IP]
```

### Verify agent enrolled

From the manager:
```bash
sudo /var/ossec/bin/agent_control -l
```

Or view in the dashboard under Agents > All Agents.

---

## Notes and Gotchas

- **RAM pressure:** Post-install memory is ~4.9 GB used of 7.6 GB. The indexer (Java/OpenSearch)
  is the heaviest consumer. If the system becomes unresponsive, add swap or reduce indexer heap:
  edit `/etc/wazuh-indexer/jvm.options` and lower `-Xms` / `-Xmx` from 1g to 512m, then restart
  the service.

- **Self-signed TLS:** The installer generates a private CA + node certs. All traffic is encrypted
  but not trusted by browsers out of the box. For production, replace with Let's Encrypt or an
  internal CA cert.

- **Firewall:** If `ufw` is active on the Ubuntu host, open the required ports:
  ```bash
  sudo ufw allow 443/tcp     # Dashboard
  sudo ufw allow 1514/tcp    # Agent events
  sudo ufw allow 1515/tcp    # Agent enrollment
  sudo ufw allow 55000/tcp   # Manager API
  ```

- **Wazuh version installed:** 4.11.2 (latest as of 2026-03-20)

- **Installer log:** `/var/log/wazuh-install.log` on the Ubuntu server for troubleshooting.

- **Config archive:** `~/wazuh-install-files.tar` on the server contains all generated certs and
  passwords. Keep this file secure.
