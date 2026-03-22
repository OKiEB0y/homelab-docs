# Homelab Walkthroughs — Master Index

**Last Updated:** 2026-03-21
**Total Walkthroughs:** 11
**Base Path:** `/home/kali/homelab-docs/walkthroughs/`

---

## Services

| Date | File | Description |
|------|------|-------------|
| 2026-03-20 | [2026-03-20_vmware-workstation-server-setup.md](services/2026-03-20_vmware-workstation-server-setup.md) | VMware Workstation Pro headless install and configuration on Ubuntu 24.04 [SIEM-HOST] ([SIEM-HOST-IP]) |
| 2026-03-20 | [2026-03-20_kali-vm-vmware-remote-deploy.md](services/2026-03-20_kali-vm-vmware-remote-deploy.md) | Deploy Kali Linux VM on remote VMware Workstation Pro from a Kali host |
| 2026-03-20 | [2026-03-20_wazuh-install.md](services/2026-03-20_wazuh-install.md) | Wazuh 4.11.2 all-in-one install (Manager + Indexer + Dashboard) on [SIEM-HOST] |
| 2026-03-21 | [2026-03-21_wazuh-agent-[SIEM-HOST].md](services/2026-03-21_wazuh-agent-[SIEM-HOST].md) | Configure Wazuh self-monitoring — enroll [SIEM-HOST] as Agent 000 reporting to itself |
| 2026-03-21 | [2026-03-21_windows-ad-lab-setup.md](services/2026-03-21_windows-ad-lab-setup.md) | Windows AD lab VM setup — WinServer2022 and Win11 deployed headless on [SIEM-HOST] via VMware |
| 2026-03-22 | [2026-03-22_server-event-monitoring.md](services/2026-03-22_server-event-monitoring.md) | Server event monitoring setup for [SIEM-HOST] ([SIEM-HOST-IP]) |

---

## Security

| Date | File | Description |
|------|------|-------------|
| 2026-03-21 | [2026-03-21_wazuh-dashboard-tls-browser-trust.md](security/2026-03-21_wazuh-dashboard-tls-browser-trust.md) | Diagnose and resolve Wazuh Dashboard "Not Secure" TLS browser warning — certificate export and trust audit |

---

## Package Management

| Date | File | Description |
|------|------|-------------|
| 2026-03-20 | [2026-03-20_impacket-installation-audit.md](package-management/2026-03-20_impacket-installation-audit.md) | Audit and verify Impacket installation on Kali Linux — confirms v0.14.0.dev0 with all 61 tools |
| 2026-03-20 | [2026-03-20_evil-winrm-installation-audit.md](package-management/2026-03-20_evil-winrm-installation-audit.md) | Audit evil-winrm installation state on Kali Linux |

---

## Networking

| Date | File | Description |
|------|------|-------------|
| 2026-03-19 | [2026-03-19_install-openvpn.md](networking/2026-03-19_install-openvpn.md) | Install and configure OpenVPN client on Kali Linux |

---

## Troubleshooting

| Date | File | Description |
|------|------|-------------|
| 2026-03-19 | [2026-03-19_bloodhound-neo4j-frozen-diagnosis.md](troubleshooting/2026-03-19_bloodhound-neo4j-frozen-diagnosis.md) | Diagnose and resolve BloodHound CE frozen state caused by Neo4j startup failure |

---

## Notes

- Source walkthroughs live at `/home/kali/linux_walkthroughs/` (original location, categorized subdirs)
- This directory (`/home/kali/homelab-docs/walkthroughs/`) is the organized reference copy
- `kerbrute_install.md` is at `/home/kali/linux_walkthroughs/kerbrute_install.md` (uncategorized — not yet migrated)
- All walkthroughs follow the standard template: Overview, Prerequisites, Pre-Change State, Steps, Verification, Rollback, Notes
