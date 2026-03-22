# Install OpenVPN on Kali Linux
**Date:** 2026-03-19
**Category:** networking
**System:** Kali Linux (6.12.25-amd64)
**Risk Level:** Low

## Overview
This walkthrough documents the verification and status check of OpenVPN on this Kali Linux machine.
OpenVPN was found to be **already installed** (version 2.6.14-1) along with NetworkManager integration
plugins. No installation was required. The walkthrough records the pre-existing state, package details,
service configuration, and next-step guidance for connecting to a VPN.

---

## Prerequisites
- Root or sudo privileges
- Internet access (only required if installation is needed)
- `apt` package manager

---

## Pre-Change State / Backup

No configuration changes were made. The system was audited in read-only fashion.

**Packages found installed:**

| Package                        | Version     | Architecture |
|-------------------------------|-------------|--------------|
| openvpn                        | 2.6.14-1    | amd64        |
| network-manager-openvpn        | 1.12.0-2    | amd64        |
| network-manager-openvpn-gnome  | 1.12.0-2    | amd64        |

**Binary location:** `/usr/sbin/openvpn`

**Config directory contents (`/etc/openvpn/`):**
```
client/
server/
update-resolv-conf
```

**Systemd service state:** `inactive (dead)` — this is expected; the base `openvpn.service`
is a oneshot/target-style unit. Per-profile services (`openvpn@<profile>.service`) activate
only when a `.conf` file is placed in `/etc/openvpn/client/` or `/etc/openvpn/server/`.

---

## Step-by-Step Instructions

### Step 1 — Check if OpenVPN is installed

```bash
dpkg -l | grep -i openvpn
which openvpn
openvpn --version
```

**Result:** All three commands confirmed OpenVPN 2.6.14 is present.

### Step 2 — Check service status

```bash
systemctl status openvpn
systemctl is-enabled openvpn
```

**Result:**
- Status: `inactive (dead)` — normal for a system with no active VPN profile loaded
- Enabled: `disabled` — the base unit is intentionally disabled; per-connection units are used instead

### Step 3 — (Skipped) Package installation

Installation was skipped because OpenVPN was already present. If it had not been installed,
the procedure would have been:

```bash
# Backup apt sources state
cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%Y%m%d_%H%M%S)

# Update package index and install
apt update && apt install -y openvpn

# Verify
openvpn --version
```

---

## Verification

```bash
# Confirm binary exists and is executable
which openvpn
# Expected: /usr/sbin/openvpn

# Confirm version and compile-time features
openvpn --version
# Expected output (confirmed):
# OpenVPN 2.6.14 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] [DCO]
# library versions: OpenSSL 3.5.4 30 Sep 2025, LZO 2.10

# Confirm package registration
dpkg -l openvpn
# Expected: ii  openvpn  2.6.14-1  amd64
```

All three checks passed. Installation is clean and functional.

---

## Connecting to a VPN (Next Steps)

### Using a .ovpn config file (most common)

```bash
# Run interactively (prompts for credentials if required)
sudo openvpn --config /path/to/your.ovpn

# Run as a systemd-managed background service
sudo cp /path/to/your.ovpn /etc/openvpn/client/myvpn.conf
sudo systemctl start openvpn-client@myvpn
sudo systemctl enable openvpn-client@myvpn   # optional: start on boot
sudo systemctl status openvpn-client@myvpn
```

### Verifying a connected tunnel

```bash
# Check for tun0 interface
ip addr show tun0

# Confirm traffic is routing through VPN
curl -s https://ifconfig.me
```

---

## Rollback / Recovery

Since no changes were made to this system, no rollback is needed.

If OpenVPN was installed fresh and needs to be removed:

```bash
# Remove package and config files
sudo apt purge --autoremove openvpn

# Optionally remove NetworkManager plugins
sudo apt purge --autoremove network-manager-openvpn network-manager-openvpn-gnome

# Verify removal
dpkg -l | grep openvpn
which openvpn
```

---

## Notes & Gotchas

- **DCO (Data Channel Offload):** The version output shows `DCO version: N/A`. DCO offloads
  encryption to the kernel for higher throughput. It requires kernel module `ovpn-dco-v2`.
  Check availability with: `modinfo ovpn-dco-v2`

- **DNS leak risk:** When connecting with `--config`, DNS may not be updated automatically.
  The `/etc/openvpn/update-resolv-conf` script (present on this system) handles this.
  Ensure your `.ovpn` file includes:
  ```
  script-security 2
  up /etc/openvpn/update-resolv-conf
  down /etc/openvpn/update-resolv-conf
  ```

- **NetworkManager integration:** With `network-manager-openvpn-gnome` installed, `.ovpn`
  files can be imported directly via the GNOME network settings GUI as an alternative to
  CLI usage.

- **Split tunneling vs. full tunnel:** By default, most `.ovpn` configs redirect all traffic
  (`redirect-gateway def1`). Confirm your config's intent before connecting on a shared network.

- **Kali-specific note:** Kali does not start OpenVPN on boot by default. This is intentional
  for a penetration testing distribution — activate VPN connections deliberately and explicitly.
