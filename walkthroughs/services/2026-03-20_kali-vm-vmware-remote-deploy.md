# Kali Linux VM Deployment on Remote VMware Workstation Pro
**Date:** 2026-03-20
**Category:** services
**System:** Ubuntu 24.04 ([SIEM-HOST] / [SIEM-HOST-IP]) — VMware Workstation Pro 17.6.3
**Risk Level:** Low

## Overview
Deploy a pre-built Kali Linux 2025.4 VMware image on a headless Ubuntu 24.04 server running
VMware Workstation Pro 17.6.3. The VM runs headlessly (nogui) and is reachable via SSH
on the VMware NAT network (172.16.126.0/24).

## Prerequisites
- SSH access to [SIEM-HOST-IP] as [admin-user] (sudo capable)
- VMware Workstation Pro 17.6.3 already installed on the remote host
- vmrun v1.17.0 available at /usr/bin/vmrun
- Packages: wget, p7zip-full (7zip)
- ~20GB free disk space (image is 3.6GB compressed, 15.7GB extracted)

## Pre-Change State / Backup
```bash
# Disk check before setup
ssh [admin-user]@[SIEM-HOST-IP] "df -h ~"
# Result: 436G available on /dev/nvme0n1p2 (/)

# No VMs were running prior
ssh [admin-user]@[SIEM-HOST-IP] "vmrun list"
# Result: Total running VMs: 0
```

## Step-by-Step Instructions

### Step 1 — Install required tools
```bash
ssh -o StrictHostKeyChecking=no [admin-user]@[SIEM-HOST-IP] \
  "echo '[REDACTED]' | sudo -S apt install -y wget p7zip-full"
# wget was already installed; 7zip + p7zip-full (transitional) were installed fresh
```

### Step 2 — Identify the latest Kali VMware image filename
```bash
ssh [admin-user]@[SIEM-HOST-IP] \
  "wget -q -O - https://cdimage.kali.org/current/ | grep -o 'kali-linux-[^\"]*vmware-amd64\.7z' | head -1"
# Returns: kali-linux-2025.4-vmware-amd64.7z
```

### Step 3 — Download the image
```bash
ssh [admin-user]@[SIEM-HOST-IP] "mkdir -p ~/VMs"
ssh [admin-user]@[SIEM-HOST-IP] "
  cd ~/VMs
  nohup wget -c --progress=dot:giga \
    https://cdimage.kali.org/current/kali-linux-2025.4-vmware-amd64.7z \
    -o /tmp/kali_wget.log > /dev/null 2>&1 &
"
# Download: 3.6GB at ~55 MB/s — completed in ~75 seconds
# File: /home/[admin-user]/VMs/kali-linux-2025.4-vmware-amd64.7z
```

### Step 4 — Extract the archive
```bash
ssh [admin-user]@[SIEM-HOST-IP] "
  cd ~/VMs
  7z x kali-linux-2025.4-vmware-amd64.7z -o.
"
# Extracted 43 files, 15.7GB uncompressed
# Output dir: /home/[admin-user]/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/
# VMX path:   /home/[admin-user]/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx
```

### Step 5 — Resolve missing VMware shared library dependencies
The vmware-vmx binary requires several X11 libs even in nogui mode. On a fresh Ubuntu
24.04 server these are missing. Identified with `ldd` and installed:

```bash
ssh [admin-user]@[SIEM-HOST-IP] "ldd /usr/lib/vmware/bin/vmware-vmx | grep 'not found'"
# Missing: libXi.so.6, libXinerama.so.1, libXcursor.so.1

ssh [admin-user]@[SIEM-HOST-IP] \
  "echo '[REDACTED]' | sudo -S apt install -y libxi6 libxtst6 libxrender1 libxinerama1 libxcursor1"
```

**Note:** VMware Workstation requires these X11 libs even for headless/nogui operation.
This is a known quirk of VMware on Ubuntu server installs.

### Step 6 — Start the VM headlessly
```bash
VMX='/home/[admin-user]/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx'
ssh [admin-user]@[SIEM-HOST-IP] "vmrun -T ws start \"$VMX\" nogui"
# Exit code: 0 (success)
# Warning about libxml 209 vs 212 is benign — version mismatch only, no functional impact
```

### Step 7 — Verify running and get guest IP
```bash
# Confirm running
ssh [admin-user]@[SIEM-HOST-IP] "vmrun list"
# Total running VMs: 1

# Wait ~75 seconds for guest boot, then fetch IP
ssh [admin-user]@[SIEM-HOST-IP] "vmrun getGuestIPAddress \"$VMX\" -wait"
# Result: [KALI-VM-IP]
```

## Verification
```bash
# VM in running list
vmrun list
# -> Total running VMs: 1

# VMware Tools running in guest
vmrun checkToolsState "$VMX"
# -> running

# Guest IP assigned
vmrun getGuestIPAddress "$VMX"
# -> [KALI-VM-IP]

# Ping from host to guest (0% packet loss)
ping -c 2 [KALI-VM-IP]
# -> 64 bytes from [KALI-VM-IP]: icmp_seq=1 ttl=64 time=0.419 ms
```

## Rollback / Recovery

### Stop and remove the VM
```bash
VMX='/home/[admin-user]/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx'

# Graceful shutdown
ssh [admin-user]@[SIEM-HOST-IP] "vmrun -T ws stop \"$VMX\" soft"

# Or hard power-off
ssh [admin-user]@[SIEM-HOST-IP] "vmrun -T ws stop \"$VMX\" hard"

# Delete VM files to reclaim ~15.7GB
ssh [admin-user]@[SIEM-HOST-IP] "rm -rf ~/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm"

# Optionally remove the 7z archive too (~3.6GB)
ssh [admin-user]@[SIEM-HOST-IP] "rm ~/VMs/kali-linux-2025.4-vmware-amd64.7z"
```

### Remove installed packages (if desired)
```bash
ssh [admin-user]@[SIEM-HOST-IP] \
  "echo '[REDACTED]' | sudo -S apt remove --purge -y p7zip-full 7zip libxi6 libxtst6 libxrender1 libxinerama1 libxcursor1 && sudo apt autoremove --purge -y"
```

## Notes & Gotchas

1. **libxml version mismatch warning** — `Warning: program compiled against libxml 212 using older 209`
   is benign. VMware was compiled against a newer libxml2 version than what Ubuntu 24.04 ships.
   All vmrun operations function correctly.

2. **VMware X11 library requirement for nogui** — VMware Workstation requires X11 libraries
   (libXi, libXinerama, libXcursor, libXtst, libXrender) even when starting VMs in nogui mode.
   On Ubuntu Server this requires: `apt install libxi6 libxtst6 libxrender1 libxinerama1 libxcursor1`

3. **VMware Tools timing** — `getGuestIPAddress` will fail if called too early. Use the `-wait`
   flag to block until Tools reports an IP, or poll manually. First boot typically takes 60-90s.

4. **Default Kali credentials** — The pre-built VMware image uses `[REDACTED]` / `[REDACTED]`.
   SSH into the guest: `ssh [REDACTED]@[KALI-VM-IP]`
   Change the password immediately after first login: `passwd`

5. **Network topology** — Kali VM is on VMware NAT (172.16.126.0/24). It can reach the internet
   through the Ubuntu host's NAT, but is not directly reachable from the LAN unless port
   forwarding or bridged networking is configured.

6. **Persistent VM start on host reboot** — The VM will NOT auto-start if the Ubuntu host reboots.
   To autostart, add a systemd service or cron @reboot entry:
   ```bash
   # Example cron entry on [admin-user]@[SIEM-HOST-IP]:
   @reboot vmrun -T ws start /home/[admin-user]/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx nogui
   ```

7. **Snapshot before first use** — Recommended before any modifications:
   ```bash
   vmrun -T ws snapshot "$VMX" "clean-install-2026-03-20"
   ```

## Final State
| Item | Value |
|------|-------|
| Host | [SIEM-HOST-IP] (Ubuntu 24.04, [admin-user]) |
| VMware Workstation | 17.6.3, vmrun 1.17.0 |
| Kali Version | 2025.4 (VMware pre-built) |
| VMX Path | /home/[admin-user]/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx |
| Guest IP | [KALI-VM-IP] (VMware NAT) |
| Guest Credentials | [REDACTED] / [REDACTED] (change immediately) |
| Disk Used (host) | 27G / 468G (6%) after full setup |
