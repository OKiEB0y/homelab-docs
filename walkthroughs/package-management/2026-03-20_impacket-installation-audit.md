# Impacket Installation Audit and Verification
**Date:** 2026-03-20
**Category:** package-management
**System:** Kali Linux (6.12.25-amd64)
**Risk Level:** Low (read-only audit; no changes made)

## Overview
Audit of impacket installation state on this Kali Linux system. Impacket was found to be
fully installed and operational via apt — no additional installation steps were required.
This walkthrough documents how to check, install, upgrade, and verify impacket on Kali Linux.

## Prerequisites
- Root or sudo access (for apt operations)
- Active internet/apt repository access for package installs or upgrades
- `python3`, `pip3`, and optionally `pipx` present

---

## Pre-Change State / Current Installation State

### apt packages installed
```
ii  python3-impacket   0.12.0+gite61ff5d-0kali1   Python3 module (system library)
ii  impacket-scripts   1.10                        Symlinks for all 61 impacket CLI tools
ii  bloodhound.py      1.8.0-0kali1                BloodHound ingestor (depends on impacket)
```

### Actual module version on disk
```
pip show impacket
# Name: impacket
# Version: 0.13.0.dev0
# Location: /usr/lib/python3/dist-packages
```
The dpkg version string (0.12.0+gite61ff5d) reflects the Kali packaging epoch, but the
actual code on disk is the 0.13.0.dev0 git snapshot — both names refer to the same files.

### apt candidate (upgrade available)
```
apt-cache policy python3-impacket
# Installed: 0.12.0+gite61ff5d-0kali1
# Candidate: 0.13.0-1
```
A clean upstream release (0.13.0-1) is available in kali-rolling. The installed git
snapshot is functionally equivalent or newer, but the apt-packaged release is cleaner.

### Tools in PATH
All 61 impacket tools are symlinked to `/usr/bin/impacket-*` pointing to
`/usr/share/impacket/script`. Key examples confirmed in PATH:
- `/usr/bin/impacket-smbclient`
- `/usr/bin/impacket-secretsdump`
- `/usr/bin/impacket-psexec`

### Reverse dependencies (packages that require impacket)
netexec, certipy-ad, bloodhound.py, smbmap, crackmapexec, lsassy, dploot, masky,
patator, polenum, spraykatz, openvas-scanner, and more.

---

## How to Install Impacket on Kali Linux (Reference)

### Method 1 — apt (preferred on Kali)
```bash
sudo apt update
sudo apt install python3-impacket impacket-scripts -y
```
This installs the system Python module plus all 61 CLI tool symlinks.
Always prefer this on Kali — it integrates cleanly with dependent packages.

### Method 2 — apt upgrade to latest release
If a newer candidate is available (check with `apt-cache policy python3-impacket`):
```bash
sudo apt update
sudo apt install --only-upgrade python3-impacket -y
```

### Method 3 — pipx (isolated, latest upstream)
Use this if you need the absolute latest version from GitHub without touching system Python:
```bash
pipx install impacket
# Tools land in ~/.local/bin/impacket-* (pipx-managed venv)
```
Note: pipx tools will NOT conflict with the apt-installed ones if PATH is ordered correctly.

### Method 4 — pip with --break-system-packages (last resort)
```bash
# WARNING: can break apt-managed packages that depend on impacket
pip install impacket --break-system-packages
```
Avoid this on Kali. The apt method is always safer.

### Method 5 — pip from GitHub (bleeding edge)
```bash
git clone https://github.com/fortra/impacket.git /opt/impacket
cd /opt/impacket
pip install . --break-system-packages
# Or use pipx: pipx install /opt/impacket
```

---

## Verification

### 1. Check dpkg package records
```bash
dpkg -l python3-impacket impacket-scripts
```
Expected: `ii` status for both packages.

### 2. Verify Python module import and version
```bash
python3 -c "import impacket; print('impacket version:', impacket.__version__)"
```

### 3. Verify via pip
```bash
pip show impacket
```

### 4. Smoke test CLI tools
```bash
impacket-smbclient --help
impacket-secretsdump --help
impacket-GetNPUsers --help     # Kerberos AS-REP roasting
impacket-getTGT --help         # Kerberos TGT request
```
Each should print `Impacket vX.Y.Z - Copyright Fortra, LLC` as the first line.

### 5. Count available tools
```bash
ls /usr/bin/impacket-* | wc -l
# Expected: 61
```

### Verification output from this session
```
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

usage: smbclient.py [-h] [-inputfile INPUTFILE] ...
```
Both `impacket-smbclient` and `impacket-secretsdump` returned correct help output. PASS.

---

## Rollback / Recovery

### Remove apt-installed impacket
```bash
sudo apt remove python3-impacket impacket-scripts --purge
sudo apt autoremove --purge
```
WARNING: This will break dependent packages (netexec, certipy-ad, bloodhound.py, smbmap,
crackmapexec, lsassy, dploot, masky, patator, polenum). Only remove if intentionally
replacing the apt install with a pipx or virtual-environment install.

### Remove pipx-installed impacket (if installed via pipx)
```bash
pipx uninstall impacket
```

### Restore apt-managed state after pip --break-system-packages install
```bash
# Force reinstall from apt to overwrite pip-placed files
sudo apt install --reinstall python3-impacket impacket-scripts -y
```

---

## Notes & Gotchas

1. **Version string discrepancy**: `dpkg -l` shows `0.12.0+gite61ff5d` but `pip show`
   shows `0.13.0.dev0`. Both refer to the same files at
   `/usr/lib/python3/dist-packages/impacket`. The Kali package version string uses the
   git hash suffix as its epoch identifier — the actual code is the 0.13.0 dev snapshot.

2. **Two parallel versions of impacket are possible**: If you install via pipx alongside
   the apt version, tools from pipx land in `~/.local/bin/` and take PATH priority if
   `~/.local/bin` is listed before `/usr/bin`. Be intentional about which version runs.

3. **Do not use pip --break-system-packages on Kali**: Kali's Python environment is
   tightly integrated. Overwriting system packages with pip can silently break tools like
   netexec, certipy-ad, and crackmapexec which all depend on the apt-managed impacket.

4. **impacket-scripts is a separate package**: The Python library (`python3-impacket`)
   and the CLI symlinks (`impacket-scripts`) are separate apt packages. Install both.
   Without `impacket-scripts`, `python3 /usr/share/impacket/script` still works but the
   `/usr/bin/impacket-*` shortcuts won't exist.

5. **Script symlink target**: All `/usr/bin/impacket-*` symlinks point to
   `/usr/share/impacket/script` (a single dispatcher script that reads `$0` to determine
   which tool to run — similar to busybox).

6. **Upgrade path**: When `apt-cache policy` shows a newer candidate, upgrade with
   `sudo apt install --only-upgrade python3-impacket` rather than a full `apt upgrade`
   to avoid pulling in unrelated updates.
