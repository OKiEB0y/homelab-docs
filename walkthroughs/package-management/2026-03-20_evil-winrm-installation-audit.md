# evil-winrm Installation Audit
**Date:** 2026-03-20
**Category:** package-management
**System:** Kali Linux
**Risk Level:** Low

## Overview
Audit whether evil-winrm is installed on the system. If absent, install via `apt`; fall back to
`gem install` if apt does not carry the package. Document the verified state and any available
upgrade path.

## Prerequisites
- `sudo` / root privileges (required for apt or gem installs)
- Internet access if installation is needed
- Ruby runtime (evil-winrm is a Ruby gem; the apt package pulls it automatically)

## Pre-Change State / Backup
No system config files are modified by this tool. No backup required.

State captured before audit:

```
$ which evil-winrm
/usr/bin/evil-winrm

$ apt list --installed 2>/dev/null | grep evil-winrm
evil-winrm/now 3.7-0kali1 all [installed,upgradable to: 3.7-0kali2]

$ evil-winrm --version
v3.7
```

## Step-by-Step Instructions

### Step 1 — Check if evil-winrm is already present

```bash
which evil-winrm
apt list --installed 2>/dev/null | grep evil-winrm
evil-winrm --version
```

**Result on this system:** Already installed as package `evil-winrm 3.7-0kali1` via apt.
Binary lives at `/usr/bin/evil-winrm`. No installation action was required.

---

### Step 2 — Install via apt (if not present)

Only execute this block if Steps above show the tool is missing:

```bash
sudo apt update && sudo apt install evil-winrm -y
```

Verify after:

```bash
which evil-winrm && evil-winrm --version
```

---

### Step 3 — Install via gem (fallback if apt has no package)

Use this only if `apt install evil-winrm` fails with "package not found":

```bash
# Ensure Ruby and build tools are present
sudo apt install ruby ruby-dev build-essential -y

# Install the gem system-wide
sudo gem install evil-winrm

# Verify
evil-winrm --version
```

The gem installs the binary to `/usr/local/bin/evil-winrm` (may vary by Ruby/gem prefix).

---

### Step 4 — Upgrade to latest available version (optional)

An upgrade from `3.7-0kali1` to `3.7-0kali2` was available at audit time:

```bash
sudo apt update && sudo apt install --only-upgrade evil-winrm -y
evil-winrm --version
```

## Verification

```bash
# Binary is reachable in PATH
which evil-winrm
# Expected: /usr/bin/evil-winrm

# Version reported cleanly
evil-winrm --version
# Expected: v3.7

# apt shows the package as installed
apt list --installed 2>/dev/null | grep evil-winrm
# Expected: evil-winrm/... all [installed...]

# Full dpkg metadata
dpkg -s evil-winrm | grep -E 'Status|Version|Depends'
# Expected: Status: install ok installed
```

## Rollback / Recovery

evil-winrm is a pentesting tool with no persistent system daemons or config files. Removal is
clean:

```bash
# If installed via apt:
sudo apt remove --purge evil-winrm -y
sudo apt autoremove --purge -y

# If installed via gem:
sudo gem uninstall evil-winrm

# Confirm removal:
which evil-winrm   # should return nothing
```

## Notes & Gotchas

- **Upgrade available:** `3.7-0kali2` is in the Kali repos. Run `sudo apt install --only-upgrade evil-winrm -y` when ready.
- **Ruby dependency:** evil-winrm depends on `ruby-winrm` and `ruby-winrm-fs`. The apt package
  handles this automatically; the gem path requires manual Ruby setup.
- **Default port:** WinRM listens on TCP 5985 (HTTP) or 5986 (HTTPS). Use `-S` flag for SSL.
- **Common usage pattern:**
  ```bash
  evil-winrm -i <target-ip> -u <username> -p <password>
  # With hash (pass-the-hash):
  evil-winrm -i <target-ip> -u <username> -H <ntlm-hash>
  # With SSL:
  evil-winrm -i <target-ip> -u <username> -p <password> -S
  ```
- **Homepage / upstream:** https://github.com/Hackplayers/evil-winrm
- **Kali maintainer:** Kali Developers <devel@kali.org>
