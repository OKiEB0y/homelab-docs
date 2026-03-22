# Wazuh Self-Monitoring: Manager Host as Agent 000
**Date:** 2026-03-21
**Category:** services
**System:** Ubuntu 24.04 LTS ([SIEM-HOST], [SIEM-HOST-IP])
**Risk Level:** Medium

## Overview
Configure the Wazuh manager host ([SIEM-HOST]) to self-monitor — i.e., the host running
wazuh-manager also reports telemetry into the Wazuh stack as an enrolled agent. This
walkthrough documents the correct architectural approach and an incident recovery after
an incorrect approach was attempted first.

## Architecture Clarification — Why You Cannot Install wazuh-agent on the Manager Host

`wazuh-agent` and `wazuh-manager` have a **hard dpkg conflict** declared in their package
metadata:

```
wazuh-agent: Conflicts: wazuh-manager
wazuh-manager: Conflicts: wazuh-agent
```

Running `apt install wazuh-agent` on a host with `wazuh-manager` installed will silently
remove wazuh-manager. This is by design: the wazuh-manager package **already includes the
full agent subsystem internally**. The manager runs its own log collectors, syscheck,
execd, and analysisd daemons — it monitors itself.

### The Correct Architecture
In Wazuh, **Agent ID 000 is always the manager itself**. It is created automatically when
wazuh-manager starts. Agent 000 is "Active/Local" — it does not communicate over the
network because it IS the manager process. All manager-side daemons (wazuh-logcollector,
wazuh-syscheckd, wazuh-modulesd, etc.) report locally into wazuh-analysisd, which indexes
events into wazuh-indexer exactly like a remote agent would.

No separate wazuh-agent installation is needed or possible on the manager host.

## Prerequisites
- wazuh-manager installed and running
- SSH access to [SIEM-HOST] ([SIEM-HOST-IP])
- sudo privileges

## Pre-Change State / Backup
```bash
# Pre-incident state
dpkg -l wazuh-manager    # ii  wazuh-manager  4.11.2-1
systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard
# all: active
```

## Incident: wazuh-manager Removed by wazuh-agent Install

### What Happened
Following the original task instructions (`apt install -y wazuh-agent`) caused apt to
resolve the conflict by removing wazuh-manager 4.11.2. This took down the manager while
leaving wazuh-indexer and wazuh-dashboard running (they are independent packages).

### Impact
- wazuh-manager.service: not-found / dead
- wazuh-indexer: still active (no data ingest)
- wazuh-dashboard: still active (UI accessible but no live data)
- Agents ([WORKSTATION]): lost connection to manager

### Recovery Steps

**Step 1 — Remove the conflicting wazuh-agent:**
```bash
echo 'PASSWORD' | sudo -S apt-get remove -y wazuh-agent
```

**Step 2 — Reinstall wazuh-manager at the pinned version:**
```bash
echo 'PASSWORD' | sudo -S apt-get install -y wazuh-manager=4.11.2-1
```

**Step 3 — Start the manager:**
```bash
echo 'PASSWORD' | sudo -S systemctl start wazuh-manager
```

**Step 4 — Verify all three services are active:**
```bash
echo 'PASSWORD' | sudo -S systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard
# Expected: active / active / active
```

## Self-Monitoring Verification (No Additional Steps Required)

Agent 000 is present and active immediately after the manager starts:

```bash
sudo /var/ossec/bin/agent_control -l
# Wazuh agent_control. List of available agents:
#    ID: 000, Name: [SIEM-HOST] (server), IP: 127.0.0.1, Active/Local

sudo /var/ossec/bin/agent_control -i 000
# Agent ID:   000 (local instance)
# Agent Name: [SIEM-HOST]
# IP address: 127.0.0.1
# Status:     Active/Local
# OS:         Linux | [SIEM-HOST] | 6.8.0-100-generic | x86_64
# Client ver: Wazuh v4.11.2
# Syscheck last started at: Sat Mar 21 17:00:14 2026
# Syscheck last ended at:   Sat Mar 21 17:00:17 2026
```

### Wazuh API Confirmation
```bash
TOKEN=$(curl -s -k -u wazuh:[REDACTED] -X POST \
  'https://localhost:55000/security/user/authenticate?raw=true')

curl -s -k -H "Authorization: Bearer $TOKEN" \
  'https://localhost:55000/agents' | python3 -m json.tool
```

**Response (relevant section):**
```json
{
  "id": "000",
  "name": "[SIEM-HOST]",
  "status": "active",
  "ip": "127.0.0.1",
  "registerIP": "127.0.0.1",
  "version": "Wazuh v4.11.2",
  "os": {
    "name": "Ubuntu",
    "version": "24.04.4 LTS",
    "platform": "ubuntu"
  },
  "lastKeepAlive": "9999-12-31T23:59:59+00:00"
}
```

The `lastKeepAlive` value of `9999-12-31` indicates "always alive / local" — this is the
expected value for the manager's built-in agent 000.

## Package Hold Applied
To prevent future `apt upgrade` runs from accidentally triggering the conflict:

```bash
sudo apt-mark hold wazuh-manager
# wazuh-manager set on hold.

apt-mark showhold
# wazuh-manager
```

To release the hold before intentional upgrades:
```bash
sudo apt-mark unhold wazuh-manager
```

## Rollback / Recovery
If wazuh-manager is removed accidentally again:

```bash
# Remove conflicting package
sudo apt-get remove -y wazuh-agent

# Reinstall exact version
sudo apt-get install -y wazuh-manager=4.11.2-1

# Start and enable
sudo systemctl start wazuh-manager
sudo systemctl enable wazuh-manager

# Re-apply hold
sudo apt-mark hold wazuh-manager

# Verify
sudo systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard
sudo /var/ossec/bin/agent_control -l
```

## Notes & Gotchas

1. **apt does not warn you** before removing wazuh-manager when installing wazuh-agent.
   It resolves the conflict silently and proceeds. Always check the "WILL BE REMOVED"
   section of apt output before confirming.

2. **Agent 000 is not listed by `manage_agents -l`** — that tool only lists enrolled
   remote agents. Use `agent_control -l` or the API to see agent 000.

3. **wazuh-indexer and wazuh-dashboard survive manager removal** because they are
   independent packages with no strict runtime dependency on the manager process being up.

4. **Wazuh manager self-monitoring is comprehensive**: wazuh-logcollector reads all
   configured log sources on the local host, wazuh-syscheckd runs FIM on local paths,
   and wazuh-modulesd handles SCA, vulnerability detection, and osquery on the local host.
   Agent 000 telemetry is functionally identical to a remote agent's telemetry.

5. **API default credentials**: wazuh/[REDACTED] (change these in production).

6. **[WORKSTATION] (Agent 001)**: Windows 11 Pro agent at [WORKSTATION-IP], was also confirmed
   active after manager recovery with lastKeepAlive timestamp updating normally.
