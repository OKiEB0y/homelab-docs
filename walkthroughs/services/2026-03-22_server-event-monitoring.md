# Server Event Monitoring — [SIEM-HOST] ([SIEM-HOST-IP])
**Date:** 2026-03-22
**Category:** services
**System:** Ubuntu 24.04.4 LTS ([SIEM-HOST], [SIEM-HOST-IP])
**Risk Level:** Low

---

## Overview

This walkthrough documents the setup of a persistent server event monitoring system on the Ubuntu 24.04 server at [SIEM-HOST-IP]. The system logs boot events, shutdown events, and apt/dpkg operations to a central log file at `/var/log/server-events.log`. This provides a lightweight audit trail of system lifecycle events without requiring a full SIEM agent on the host itself.

---

## What Was Set Up

### Log File
- `/var/log/server-events.log` — central event log, permissions `644`, owned by root

### Scripts (in `/usr/local/bin/`)
| Script | Purpose |
|--------|---------|
| `log-boot-event.sh` | Runs at every boot; logs kernel, IP, uptime, VMware VM count, and Wazuh service states |
| `log-shutdown-event.sh` | Runs at shutdown/reboot; logs timestamp and initiating user |
| `log-upgrade-event.sh` | Called by dpkg hook after apt operations; logs package count and last apt history entry |
| `log-apt-update-event.sh` | Called by apt hook after `apt update`; logs timestamp with APT UPDATE tag |

### Systemd Services (in `/etc/systemd/system/`)
| Service | Trigger | Status |
|---------|---------|--------|
| `boot-logger.service` | `multi-user.target` (every boot) | enabled, active (exited) |
| `shutdown-logger.service` | `halt.target reboot.target shutdown.target` | enabled |

### APT Hooks (in `/etc/apt/apt.conf.d/`)
| File | Hook | Action |
|------|------|--------|
| `99-update-logger` | `DPkg::Post-Invoke` | Calls `log-upgrade-event.sh` after every dpkg operation |
| `99-update-logger` | `APT::Update::Post-Invoke-Success` | Calls `log-apt-update-event.sh` after successful `apt update` |

---

## How to Read the Event Log

```bash
sudo cat /var/log/server-events.log
```

**Event types by prefix/delimiter:**
- `=== ... ===` blocks — boot or shutdown events (structured, multi-line)
- `--- ... ---` blocks — upgrade/dpkg events (structured, multi-line)
- `YYYY-MM-DD HH:MM:SS UTC | ...` lines — single-line timestamped events (manual or apt update)

**Example output:**
```
========================================
BOOT EVENT: 2026-03-22 02:04:07 UTC
Kernel: 6.8.0-100-generic
IP Address: [SIEM-HOST-IP]
Uptime: up 1 day, 50 minutes
VMware VMs: Total running VMs: 0
wazuh-manager: active
wazuh-indexer: active
wazuh-dashboard: active
========================================
2026-03-22 02:04:08 UTC | MANUAL REBOOT INITIATED by admin
========================================
BOOT EVENT: 2026-03-22 02:04:48 UTC
Kernel: 6.8.0-106-generic
IP Address: [SIEM-HOST-IP]
Uptime: up 0 minutes
VMware VMs: Total running VMs: 0
wazuh-manager: activating
wazuh-indexer: activating
wazuh-dashboard: failed
========================================
```

**Note on the boot event snapshot above:** The wazuh-dashboard showed `failed` at T+0 seconds because the boot-logger ran before the network interface was fully up. The dashboard requires `[SIEM-HOST-IP]:443` to be bindable — this was fixed with a systemd drop-in (see below). By ~T+90s all services were active.

---

## Notable Issues Resolved During Setup

### 1. wazuh-dashboard Fails at Boot (EADDRNOTAVAIL)
**Problem:** `wazuh-dashboard` binds to `[SIEM-HOST-IP]:443`. At boot, it starts before the network interface has been assigned that IP, causing `EADDRNOTAVAIL`.

**Fix:** Created a systemd drop-in override at `/etc/systemd/system/wazuh-dashboard.service.d/network-wait.conf`:
```ini
[Unit]
After=network-online.target wazuh-indexer.service
Wants=network-online.target

[Service]
Restart=on-failure
RestartSec=15s
StartLimitBurst=5
StartLimitIntervalSec=120
```
This ensures the dashboard waits for the network and auto-restarts on failure.

### 2. VMware Kernel Modules Not Built for New Kernel (6.8.0-106-generic)
**Problem:** The kernel upgraded from `6.8.0-100` to `6.8.0-106` during reboot. VMware's `vmmon` and `vmnet` modules were not compiled for the new kernel, causing the `vmware.service` to fail.

**Fix:**
```bash
sudo vmware-modconfig --console --install-all
```
This rebuilt `vmmon.ko` and `vmnet.ko` for `6.8.0-106-generic` and loaded them successfully.

**This must be repeated any time the kernel updates.** See log rotation section for automation idea.

### 3. WinServer2022 VM Crashes on Start (PCIe Slot Error)
**Problem:** VMware Workstation 17.6.3 on kernel `6.8.0-106-generic` with `virtualHW.version = "18"` fails to allocate a PCIe slot for the `e1000e` NIC, causing `vmware-vmx` to SIGSEGV.

**Root cause log entry:**
```
[msg.pci.noslotavail] No PCIe slot available for Ethernet0.
E1000PCI: failed to register e1000e device.
```

**Fix:** Changed `ethernet0.virtualDev` from `e1000e` to `e1000` (legacy PCI, not PCIe) and removed the explicit `ethernet0.pciSlotNumber` from the VMX:

```bash
# In /home/[admin-user]/VMs/WinServer2022/WinServer2022.vmx
# Change:  ethernet0.virtualDev = "e1000e"
# To:      ethernet0.virtualDev = "e1000"
# Remove:  ethernet0.pciSlotNumber = "21"
```

The `e1000` adapter uses legacy PCI bus allocation which is unaffected by the PCIe slot exhaustion bug. The VM started successfully after this change.

**Backup of original VMX:** `/home/[admin-user]/VMs/WinServer2022/WinServer2022.vmx.bak.20260322_021046`

### 4. vmrun libxml2 Version Mismatch Warning
**Non-fatal warning:** `vmrun` reports `Warning: program compiled against libxml 212 using older 209` because the system has `libxml2 2.9.14` but `vmrun` was built against `2.12.x`.

VMware ships its own libxml2 at `/usr/lib/vmware/lib/libxml2.so.2/libxml2.so.2` (version 2.12.5). The `vmrun list` command works fine. The `start` command for `e1000`-based VMs also works despite the warning.

---

## How to Add Additional Events

Add any custom event to the log from any script:
```bash
echo "$(date '+%Y-%m-%d %H:%M:%S UTC' --utc) | YOUR EVENT DESCRIPTION" | sudo tee -a /var/log/server-events.log
```

To add a new service check to boot events, edit `/usr/local/bin/log-boot-event.sh`:
```bash
MYSERVICE=$(systemctl is-active my-service 2>/dev/null)
echo "my-service: $MYSERVICE" >> $LOG
```

To trigger an event on a schedule, add a cron entry:
```bash
# Example: log disk usage at midnight
0 0 * * * /bin/bash -c 'echo "$(date +%Y-%m-%d\ %H:%M:%S\ UTC --utc) | DISK: $(df -h / | tail -1 | awk '"'"'{print $5}'"'"') used" >> /var/log/server-events.log'
```

---

## Log Rotation Setup (Recommended)

The log will grow indefinitely without rotation. Add a logrotate config:

```bash
sudo tee /etc/logrotate.d/server-events << 'EOF'
/var/log/server-events.log {
    monthly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
    postrotate
        echo "$(date '+%Y-%m-%d %H:%M:%S UTC' --utc) | LOG ROTATED" >> /var/log/server-events.log
    endscript
}
EOF
```

Test with: `sudo logrotate -d /etc/logrotate.d/server-events`

---

## Verification Commands

```bash
# View the event log
sudo cat /var/log/server-events.log

# Check boot-logger ran at last boot
systemctl status boot-logger.service

# Verify services enabled
systemctl is-enabled boot-logger.service shutdown-logger.service

# Test the boot logger manually
sudo /usr/local/bin/log-boot-event.sh && tail -10 /var/log/server-events.log

# Verify apt hooks syntax
apt-config dump | grep -A1 "Post-Invoke"

# Check kernel modules
lsmod | grep -E "vmmon|vmnet"

# Check running VMs
vmrun list
```

---

## Rollback / Recovery

### Remove the event monitoring system entirely:
```bash
sudo systemctl disable --now boot-logger.service shutdown-logger.service
sudo rm /etc/systemd/system/boot-logger.service
sudo rm /etc/systemd/system/shutdown-logger.service
sudo rm /etc/systemd/system/wazuh-dashboard.service.d/network-wait.conf
sudo rm /usr/local/bin/log-boot-event.sh
sudo rm /usr/local/bin/log-shutdown-event.sh
sudo rm /usr/local/bin/log-upgrade-event.sh
sudo rm /usr/local/bin/log-apt-update-event.sh
sudo rm /etc/apt/apt.conf.d/99-update-logger
sudo systemctl daemon-reload
```

### Restore WinServer2022 VMX to original (e1000e NIC):
```bash
sudo cp /home/[admin-user]/VMs/WinServer2022/WinServer2022.vmx.bak.20260322_021046 \
        /home/[admin-user]/VMs/WinServer2022/WinServer2022.vmx
sudo chown [admin-user]:[admin-user] /home/[admin-user]/VMs/WinServer2022/WinServer2022.vmx
```
Note: restoring the original VMX will likely cause the VM to fail to start again on kernel 6.8.0-106+.

---

## Notes & Gotchas

1. **Boot logger timing**: The boot-logger runs after `network-online.target`, so Wazuh services captured in the log reflect their state at ~boot+30s, not at the moment the log was read. Wazuh-dashboard may show `failed` or `activating` if it starts slowly.

2. **Kernel upgrades break VMware modules**: Every kernel update requires `sudo vmware-modconfig --console --install-all` to rebuild `vmmon`/`vmnet`. Consider adding this to an `apt` post-install hook for linux-image packages.

3. **e1000e vs e1000 on kernel 6.8.0-106**: The PCIe slot allocator bug in VMware Workstation 17.6.3 affects both `e1000e` and `vmxnet3` adapters. Only `e1000` (legacy PCI) works reliably. This may be fixed in a future VMware Workstation release.

4. **vmrun libxml2 warning**: The `Warning: program compiled against libxml 212 using older 209` is cosmetic on `list` and `e1000`-based starts. If future VM configs use features that depend on libxml2 parsing in the start path, this may become a real issue. Monitor VMware release notes for updates.

5. **vmware.service shows failed in systemctl**: The `vmware` init.d wrapper returns exit code 1 if any sub-service (like the authentication daemon) is already running when it checks, even though modules are loaded and VMs run fine. This is a cosmetic systemd reporting issue with init.d wrappers.

6. **APT hook format**: The `apt.conf.d` format does not support shell command substitution (`$(...)`) inline. Use script files called from the hooks instead.
