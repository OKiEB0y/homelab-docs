# Home Security Operations Lab

A self-hosted cybersecurity home lab built on bare metal, featuring a VMware hypervisor, a full Wazuh SIEM stack, and a multi-OS virtual machine environment. This lab is used for hands-on security engineering, detection engineering, and adversary simulation.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Infrastructure](#infrastructure)
- [Wazuh SIEM Deployment](#wazuh-siem-deployment)
- [Virtual Machine Environment](#virtual-machine-environment)
- [TLS & PKI Configuration](#tls--pki-configuration)
- [Monitored Endpoints](#monitored-endpoints)
- [Skills Demonstrated](#skills-demonstrated)
- [Lessons Learned](#lessons-learned)
- [Roadmap](#roadmap)

---

## Overview

This lab simulates a small enterprise environment with a centralized SIEM, a hypervisor for spinning up target and attacker VMs, and real endpoint monitoring across Windows and Linux hosts. Everything is deployed on physical hardware and managed remotely over SSH.

**Goals:**
- Build and operate a production-grade SIEM from scratch
- Practice detection engineering, log analysis, and threat hunting
- Simulate adversary techniques in an isolated environment (Active Directory attacks, lateral movement, etc.)
- Document everything to professional standard

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Home Network (192.168.1.0/24)           │
│                                                             │
│  ┌──────────────────┐          ┌─────────────────────────┐  │
│  │   Kali Linux     │          │   Devin-PC              │  │
│  │   (Attacker)     │          │   Windows 11 Pro        │  │
│  │                  │          │   192.168.1.241          │  │
│  └────────┬─────────┘          └──────────┬──────────────┘  │
│           │  SSH / Pentest                │ Wazuh Agent 001  │
│           │                               │                  │
│  ┌────────▼───────────────────────────────▼──────────────┐  │
│  │              laptopserver (192.168.1.89)               │  │
│  │              Ubuntu 24.04.4 LTS — Intel i7-10750H      │  │
│  │              12 vCPUs │ 7.6 GB RAM │ 468 GB NVMe       │  │
│  │                                                        │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │           Wazuh 4.11.2 (All-in-One)             │  │  │
│  │  │   wazuh-manager │ wazuh-indexer │ wazuh-dashboard│  │  │
│  │  │   Agent 000: self (laptopserver)                │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │                                                        │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │       VMware Workstation Pro 17.6.3              │  │  │
│  │  │                                                 │  │  │
│  │  │   ┌──────────────────────────────────────────┐  │  │  │
│  │  │   │  Kali Linux 2025.4 VM                    │  │  │  │
│  │  │   │  172.16.126.128 (VMware NAT)              │  │  │  │
│  │  │   │  BloodHound CE │ Impacket │ Neo4j         │  │  │  │
│  │  │   └──────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Infrastructure

### Hypervisor Host — laptopserver

| Component | Detail |
|-----------|--------|
| **OS** | Ubuntu 24.04.4 LTS (Noble) |
| **Hostname** | laptopserver |
| **IP** | 192.168.1.89 |
| **CPU** | Intel Core i7-10750H (6 cores / 12 threads) |
| **RAM** | 7.6 GB |
| **Storage** | 468 GB NVMe |
| **Hypervisor** | VMware Workstation Pro 17.6.3 |
| **Access** | SSH key-based authentication (ED25519) |

### Key Configuration Steps

**SSH Hardening:**
- Disabled password authentication via `/etc/ssh/sshd_config.d/50-cloud-init.conf`
- Deployed ED25519 key pair for passwordless access from the attacker machine
- Confirmed remote access before closing interactive session

**VMware Workstation Installation:**
- Installed build dependencies: `build-essential`, `linux-headers-$(uname -r)`, `libaio1t64`
- Ran headless install: `sudo ./VMware-Workstation.bundle --console --required --eulas-agreed`
- Resolved Ubuntu 24.04 compatibility — `libaio1` renamed to `libaio1t64`
- Resolved missing X11 libraries required by VMware even in `nogui` mode: `libxi6 libxtst6 libxrender1 libxinerama1 libxcursor1`
- Pinned kernel module recompilation procedure for post-upgrade recovery: `sudo vmware-modconfig --console --install-all`

---

## Wazuh SIEM Deployment

### Stack

| Component | Version | Port | Purpose |
|-----------|---------|------|---------|
| wazuh-manager | 4.11.2 | 1514, 1515, 55000 | Agent communication, rule engine, alert generation |
| wazuh-indexer | 4.11.2 (OpenSearch) | 9200 | Log storage and indexing |
| wazuh-dashboard | 4.11.2 (OpenSearch Dashboards) | 443 | Web UI, alert visualization |
| filebeat | — | — | Ships manager events to indexer |

### Deployment Method

Used the official Wazuh installation assistant for a single-node all-in-one deployment:

```bash
curl -sO https://packages.wazuh.com/4.11/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.11/config.yml
# Configure single-node topology in config.yml
sudo bash wazuh-install.sh --generate-config-files
sudo bash wazuh-install.sh --wazuh-indexer node-1
sudo bash wazuh-install.sh --start-cluster
sudo bash wazuh-install.sh --wazuh-server wazuh-1
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

### Notable Architecture Decision — Self-Monitoring

Attempted to install `wazuh-agent` on the manager host for self-monitoring. Discovered a hard dpkg `Conflicts` declaration between `wazuh-manager` and `wazuh-agent` — they cannot coexist on the same host. This is by design: **Agent 000 is always the manager host**, running in-process with permanent `Active/Local` status. All local monitoring daemons (wazuh-logcollector, wazuh-syscheckd, wazuh-modulesd) run against the host automatically.

**Recovery procedure applied:** reinstalled `wazuh-manager` pinned to `4.11.2-1` and held with `apt-mark hold wazuh-manager` to prevent future conflict-triggered removal during upgrades.

### Dashboard Access

- **URL:** `https://192.168.1.89`
- **Auth:** Username/password (auto-generated during install)
- **TLS:** Self-signed certificate issued by internal Wazuh CA (see below)

---

## TLS & PKI Configuration

### Certificate Architecture

The Wazuh installer auto-generates a full internal PKI using OpenSSL:

**Root CA Generation:**
```bash
openssl req -x509 -new -nodes -newkey rsa:2048 \
  -keyout root-ca.key \
  -out root-ca.pem \
  -batch -subj '/OU=Wazuh/O=Wazuh/L=California/' \
  -days 3650
```

**Dashboard Certificate (with IP SAN):**
```bash
# Generate CSR with SAN config
openssl req -new -nodes -newkey rsa:2048 \
  -keyout dashboard-key.pem -out dashboard.csr \
  -config dashboard.conf   # includes subjectAltName = IP:192.168.1.89

# Sign with internal CA
openssl x509 -req -in dashboard.csr \
  -CA root-ca.pem -CAkey root-ca.key -CAcreateserial \
  -out dashboard.pem -extfile dashboard.conf \
  -extensions v3_req -days 3650
```

Key detail: the `-extensions v3_req` flag is required to embed the SAN into the signed certificate. Without this, modern browsers reject the cert even if the CSR contained the SAN — a common misconfiguration in self-signed deployments.

### Browser Trust Configuration

Exported the Wazuh root CA and distributed to client machines:

```bash
scp okieb0y@192.168.1.89:/home/okieb0y/wazuh-root-ca.crt .
```

**Windows (Chrome/Edge):** Imported via `certmgr.msc` → Trusted Root Certification Authorities
**Firefox:** Imported via Settings → Privacy & Security → View Certificates → Authorities

**CA Fingerprint (SHA-256):**
```
15:91:AC:E7:1E:09:4A:C4:18:05:DD:C8:5D:48:66:E5:
A7:42:41:7D:52:1A:B8:1F:84:C8:7D:63:94:E5:56:42
```

---

## Virtual Machine Environment

### Kali Linux 2025.4 (Attacker VM)

| Detail | Value |
|--------|-------|
| **IP** | 172.16.126.128 (VMware NAT) |
| **Access** | `ssh -J okieb0y@192.168.1.89 kali@172.16.126.128` |
| **VMX** | `/home/okieb0y/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/` |

**Tooling installed:**

| Tool | Version | Purpose |
|------|---------|---------|
| Impacket | 0.14.0.dev0 | AD attack suite (secretsdump, GetNPUsers, etc.) |
| BloodHound CE | 8.7.0 | Active Directory attack path visualization |
| SharpHound | 2.9.0 | AD data collection for BloodHound |
| AzureHound | 2.9.0 | Azure AD data collection |
| Neo4j | 4.4.26 | Graph database backend for BloodHound |
| Evil-WinRM | 3.7 | WinRM shell for post-exploitation |

**VM Management (from laptopserver):**
```bash
vmrun -T ws start ~/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx nogui
vmrun -T ws stop  ~/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx
vmrun list
```

---

## Monitored Endpoints

| Agent ID | Hostname | OS | IP | Status |
|----------|----------|----|----|--------|
| 000 | laptopserver | Ubuntu 24.04.4 LTS | 127.0.0.1 (local) | Active |
| 001 | Devin-PC | Windows 11 Pro | 192.168.1.241 | Active |

---

## Skills Demonstrated

| Category | Skills |
|----------|--------|
| **Systems Administration** | Ubuntu server management, SSH hardening, systemd service management, apt package pinning, disk/memory capacity planning |
| **Virtualization** | VMware Workstation Pro headless deployment, VM lifecycle management via `vmrun` CLI, VMware NAT networking, kernel module management |
| **SIEM Engineering** | Wazuh all-in-one deployment, single-node cluster configuration, agent enrollment, log pipeline (manager → filebeat → indexer → dashboard) |
| **PKI / TLS** | OpenSSL CA creation, certificate signing with SANs, browser trust store management, certificate auditing and verification |
| **Networking** | SSH jump host configuration, VMware NAT, port scanning with nmap, multi-subnet routing |
| **Security Tooling** | BloodHound CE, Impacket suite, Evil-WinRM, Neo4j |
| **Documentation** | Architecture diagramming, runbook authoring, incident documentation |

---

## Lessons Learned

**1. `wazuh-manager` and `wazuh-agent` are mutually exclusive packages**
A hard dpkg conflict prevents both from being installed on the same host. The manager self-monitors as Agent 000 automatically — no agent installation is needed or possible on the manager host.

**2. Ubuntu 24.04 renames `libaio1` to `libaio1t64`**
VMware Workstation's dependency check fails on Ubuntu 24.04 if you attempt to install `libaio1` — the package was renamed. The correct package is `libaio1t64`.

**3. OpenSSL `-extensions` flag is required to embed SANs in signed certs**
A CSR containing a SAN will not have that SAN carried into the signed certificate unless `-extfile` and `-extensions v3_req` are explicitly passed to `openssl x509 -req`. Modern browsers (Chrome 58+) reject certificates without a SAN, making this a critical step.

**4. VMware requires X11 libraries even in `nogui` mode on Ubuntu Server**
VMware Workstation links against `libxi6`, `libxtst6`, `libxrender1`, `libxinerama1`, and `libxcursor1` even when running headless. These must be installed on a server with no desktop environment.

**5. `apt-mark hold` is essential after manual package installs**
Without holding `wazuh-manager`, a future `apt upgrade` could resolve the package conflict by silently removing the manager. Always hold manually-pinned security-critical packages.

---

## Roadmap

- [ ] Deploy Windows Server 2022 VM and configure as Active Directory Domain Controller
- [ ] Join Windows 11 VM to the domain for realistic AD environment
- [ ] Enroll Kali VM as a Wazuh agent
- [ ] Build custom Wazuh detection rules for common AD attack techniques (AS-REP roasting, Kerberoasting, Pass-the-Hash)
- [ ] Run BloodHound against the AD lab and document attack paths
- [ ] Configure Wazuh active response for automated threat containment
- [ ] Set up Wazuh email/Slack alerting
- [ ] Expand to 32 GB RAM for additional concurrent VMs

---

## Repository Structure

```
homelab-docs/
├── README.md                          # This file
└── walkthroughs/                      # Linked from /home/kali/linux_walkthroughs/
    ├── services/
    │   ├── wazuh-install.md
    │   ├── vmware-workstation-setup.md
    │   ├── kali-vm-deploy.md
    │   └── wazuh-agent-laptopserver.md
    ├── security/
    │   └── wazuh-dashboard-tls.md
    └── package-management/
        ├── impacket-audit.md
        ├── evil-winrm-audit.md
        └── kali-update-bloodhound-neo4j.md
```

---

*Built and maintained by a security practitioner focused on hands-on blue team and red team operations.*
