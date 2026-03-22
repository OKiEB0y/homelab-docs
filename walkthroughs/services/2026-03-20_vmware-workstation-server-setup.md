# VMware Workstation Pro — Headless Server Setup on Ubuntu 24.04.4 LTS
**Date:** 2026-03-20
**Category:** services
**System:** Ubuntu 24.04.4 LTS ([SIEM-HOST]) — accessed from Kali Linux
**Target Host:** [SIEM-HOST-IP] ([admin-user]@[SIEM-HOST])
**Risk Level:** Medium
**Install Status:** COMPLETED 2026-03-21 — VMware Workstation Pro 17.6.3, vmrun 1.17.0 build-24583834

---

## Overview

Configures [SIEM-HOST] (Ubuntu 24.04.4 LTS) as a VMware Workstation Pro hypervisor host
capable of running multiple VMs simultaneously, managed entirely over SSH with no physical
display. Covers: pre-flight assessment, VMware Workstation Pro installation, headless
service configuration, vmrun CLI management, and remote display access via VNC.

---

## Actual Installation Results (2026-03-21)

### What was installed
- **VMware Workstation Pro:** 17.6.3
- **vmrun (VIX):** 1.17.0 build-24583834
- **OVF Tool:** 4.6.3
- **VMware Tools (Windows guests):** 12.5.0

### Kernel modules loaded and confirmed
```
vmnet      73728  13
vmmon     163840  0
vmw_vsock_vmci_transport  49152  0
vmw_vmci  106496  1 vmw_vsock_vmci_transport
```

### Services
- `vmware.service` — **active (running)** — enabled at boot
- `vmware-USBArbitrator.service` — **masked** (expected; headless server has no USB bus)

### Key deviations from original plan
1. Bundle file (`VMware-Workstation.bundle`, 336MB) was already at `~/VMware-Workstation.bundle`
   — rename step was already completed in a prior session.
2. `libaio1` does not exist on Ubuntu 24.04 — the correct package is `libaio1t64`.
3. `vmware-networks.service` does not exist as a separate systemd unit — networking is
   handled inside `vmware.service`. Only two SysV-generated units exist:
   `vmware.service` and `vmware-USBArbitrator.service`.
4. `vmrun --version` is not a supported flag — vmrun prints version info to stdout
   alongside the full usage page and exits 1. Version confirmed as `1.17.0 build-24583834`.

---

## Pre-Flight Assessment Results (2026-03-20)

### CPU Virtualization
- **CPU:** Intel Core i7-10750H @ 2.60GHz
- **Cores/Threads:** 12 logical processors
- **VMX flag present:** YES (24 logical cores all report `vmx`)
- **KVM kernel modules loaded:** `kvm_intel`, `kvm`, `irqbypass`
- **/dev/kvm exists:** YES (`crw-rw---- 1 root kvm`)
- **Nested virtualization:** ENABLED (`/sys/module/kvm_intel/parameters/nested = Y`)

### Memory
- **Total RAM:** 7.6 GiB
- **Available:** ~7.0 GiB
- **Swap:** 4.0 GiB
- **Assessment:** Sufficient for 2-3 lightweight VMs (512MB-1GB each). Constrained for
  heavier workloads. Monitor carefully.

### Disk
- **Device:** NVMe SSD (nvme0n1) — 476.9 GB total
- **Root partition:** 468 GB, 6.7 GB used, **437 GB free (94% available)**
- **Assessment:** Excellent. Comfortably supports 5-10+ VM disk images at typical sizes
  (20-50 GB each).

### Existing Hypervisor State
- **VMware Workstation:** NOT installed
- **KVM/QEMU:** Kernel modules present (loaded by default on Ubuntu), but no userspace
  tools installed
- **VirtualBox:** NOT installed
- **open-vm-tools:** Installed (this is the *guest* agent, not the host hypervisor —
  safe to leave or remove)
- **VMware installer bundle:** NOT found in /tmp, /home/[admin-user], or Downloads

### User
- **User:** [admin-user] (uid=1000)
- **Groups:** sudo, adm, cdrom, dip, plugdev, lxd
- **Sudo access:** Yes (password required interactively)

---

## Prerequisites

- Ubuntu 24.04.4 LTS with kernel 6.8.0-100-generic
- sudo access (password required — must use interactive TTY or pre-configure NOPASSWD)
- Internet access on [SIEM-HOST] (for downloading dependencies)
- VMware Workstation Pro installer bundle (see Step 1 below)
- Approximately 1.5 GB free for installation

---

## Step 1 — Obtain VMware Workstation Pro

VMware Workstation Pro is now **free for personal use** as of May 2024 (Broadcom
acquisition). It requires a free Broadcom account.

### Download Instructions

1. On any machine with a browser, go to:
   https://support.broadcom.com/

2. Create a free account (or sign in) at:
   https://profile.broadcom.com/web/registration

3. After logging in, navigate to:
   **Support Portal > VMware Cloud Foundation > My Downloads**
   OR direct URL:
   https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware+Workstation+Pro

4. Select: **VMware Workstation Pro 17.x for Linux**
   (As of early 2026, the current release is 17.6.x)

5. Download the `.bundle` file, e.g.:
   `VMware-Workstation-Full-17.6.x-xxxxxxx.x86_64.bundle`

6. Transfer the bundle to [SIEM-HOST]:

```bash
scp VMware-Workstation-Full-17.6.x-xxxxxxx.x86_64.bundle [admin-user]@[SIEM-HOST-IP]:/tmp/
```

Or download directly on [SIEM-HOST] (replace URL with the signed download link from
Broadcom's portal — links are session-authenticated and expire):

```bash
ssh [admin-user]@[SIEM-HOST-IP]
wget -O /tmp/VMware-Workstation.bundle "BROADCOM_SIGNED_URL"
```

---

## Step 2 — Prepare the System

Run on [SIEM-HOST] via SSH:

```bash
ssh [admin-user]@[SIEM-HOST-IP]
```

### 2a. Install build dependencies

VMware requires kernel headers and build tools to compile its kernel modules:

```bash
# NOTE: On Ubuntu 24.04+, libaio1 was renamed to libaio1t64
sudo apt update && sudo apt install -y \
  build-essential \
  linux-headers-$(uname -r) \
  gcc \
  make \
  libaio1t64 \
  libssl-dev \
  libglib2.0-dev \
  git \
  wget \
  curl
```

### 2b. Verify kernel headers are present

```bash
ls /usr/src/linux-headers-$(uname -r)/
```

Should return a populated directory. If empty or missing, run:

```bash
sudo apt install -y linux-headers-$(uname -r)
```

### 2c. (Optional) Remove open-vm-tools if it causes conflicts

open-vm-tools is a *guest* agent. On a bare-metal host running VMware Workstation, it
is unnecessary and may conflict. Remove if you see issues:

```bash
sudo apt remove --purge open-vm-tools -y && sudo apt autoremove --purge -y
```

---

## Step 3 — Install VMware Workstation Pro

```bash
# Set the bundle filename (adjust to your downloaded version)
BUNDLE=/tmp/VMware-Workstation-Full-17.6.x-xxxxxxx.x86_64.bundle

# Make executable
chmod +x "$BUNDLE"

# Run the installer in console (headless) mode
sudo "$BUNDLE" --console --required --eulas-agreed
```

**Flags explained:**
- `--console` — text-only install, no GUI required
- `--required` — accept defaults for all prompts
- `--eulas-agreed` — accept EULAs non-interactively (read them beforehand)

### Expected install duration: 3-8 minutes

The installer will:
1. Extract files to `/usr/lib/vmware/`
2. Compile kernel modules (vmmon, vmnet)
3. Install `vmrun`, `vmware`, `vmplayer` binaries to `/usr/bin/`
4. Create systemd service units

### Verify installation

```bash
vmware --version
vmrun --version
ls /usr/lib/vmware/
systemctl status vmware vmware-networks vmware-usbarbitrator 2>/dev/null
```

---

## Step 4 — Configure for Headless/Server Operation

### 4a. Start and enable VMware services

NOTE: VMware on Ubuntu uses SysV init scripts wrapped by systemd. Only two units are
generated: `vmware.service` (core, required) and `vmware-USBArbitrator.service` (USB
passthrough — not applicable on headless servers). There is no separate `vmware-networks`
unit; networking is handled inside `vmware.service`.

```bash
# Enable and start the core service (SysV redirect message is normal)
sudo systemctl enable vmware.service
sudo systemctl start vmware.service

# Mask the USB arbitrator on headless servers — it will fail if not masked
sudo systemctl mask vmware-USBArbitrator.service
```

Verify:

```bash
sudo systemctl status vmware.service --no-pager
```

Expected output confirms: Virtual machine monitor, VMCI, vmxnet (virtual ethernet),
VMware Authentication Daemon, and Shared Memory all show "done".

### 4b. Configure VMware network (NAT + Host-Only)

VMware creates virtual network adapters. For headless server use:

```bash
# Initialize virtual networks
sudo vmware-networks --start

# List configured networks
sudo vmware-networks --status
```

Default networks:
- `vmnet1` — Host-Only (no external access, for isolated VMs)
- `vmnet8` — NAT (VMs share host internet via NAT)

To configure a bridged network (VMs get IPs directly on your LAN):

```bash
# Find your physical NIC
ip link show

# Add bridged network (replace enp3s0 with your actual interface)
sudo vmware-networks --configure-bridges enp3s0
```

### 4c. Kernel module verification

```bash
lsmod | grep -E 'vmmon|vmnet'
```

If modules are not loaded:

```bash
sudo modprobe vmmon
sudo modprobe vmnet
```

To persist across reboots (if not already handled by VMware services):

```bash
echo -e 'vmmon\nvmnet' | sudo tee /etc/modules-load.d/vmware.conf
```

---

## Step 5 — Managing VMs via vmrun (CLI)

`vmrun` is VMware's headless VM management tool. All operations work over SSH.

### Create a new VM directory

```bash
mkdir -p /home/[admin-user]/vmware/vm1
```

### Start a VM (no GUI)

```bash
vmrun -T ws start /home/[admin-user]/vmware/vm1/vm1.vmx nogui
```

### Stop a VM gracefully

```bash
vmrun -T ws stop /home/[admin-user]/vmware/vm1/vm1.vmx soft
```

### Hard power off

```bash
vmrun -T ws stop /home/[admin-user]/vmware/vm1/vm1.vmx hard
```

### Suspend / Resume

```bash
vmrun -T ws suspend /home/[admin-user]/vmware/vm1/vm1.vmx
vmrun -T ws reset /home/[admin-user]/vmware/vm1/vm1.vmx
```

### List running VMs

```bash
vmrun list
```

### Take a snapshot

```bash
vmrun -T ws snapshot /path/to/vm.vmx "snapshot-name"
vmrun -T ws listSnapshots /path/to/vm.vmx
vmrun -T ws revertToSnapshot /path/to/vm.vmx "snapshot-name"
```

### Run a command inside a running VM (requires VMware Tools in guest)

```bash
vmrun -T ws -gu USERNAME -gp PASSWORD runProgramInGuest /path/to/vm.vmx /bin/bash -c "whoami"
```

---

## Step 6 — Remote Display Access

Since the server is headless, two options exist for viewing VM consoles:

---

### Option A: VNC (Recommended for simplicity)

VMware Workstation supports built-in VNC per-VM. Add these lines to a VM's `.vmx` file:

```bash
# Edit the .vmx file for the target VM
# Replace /path/to/vm.vmx with the actual path
nano /path/to/vm.vmx
```

Add at the bottom of the .vmx file:

```
RemoteDisplay.vnc.enabled = "TRUE"
RemoteDisplay.vnc.port = "5901"
RemoteDisplay.vnc.password = "[REDACTED]"
```

Use a different port for each VM (5901, 5902, 5903...).

Restart the VM after editing:

```bash
vmrun -T ws stop /path/to/vm.vmx hard
vmrun -T ws start /path/to/vm.vmx nogui
```

Connect from your Kali machine:

```bash
# SSH tunnel (secure — do NOT expose VNC port directly to internet)
ssh -L 5901:localhost:5901 [admin-user]@[SIEM-HOST-IP] -N &

# Then connect your VNC client to localhost:5901
vncviewer localhost:5901
# or: gvncviewer, remmina, tigervnc, etc.
```

---

### Option B: VMware Remote Console (VMRC)

VMRC is VMware's official remote console client. It provides a richer experience than VNC.

#### Server side (on [SIEM-HOST])

VMRC requires VMware Workstation's remote access feature to be enabled. After installing
Workstation Pro:

```bash
# Enable shared VMs / remote access
sudo vmware-wssc-adminTool setSharedVMsLocation /home/[admin-user]/vmware/shared
sudo systemctl enable --now vmware-workstation-server
```

#### Client side (on your Kali machine or Windows/Mac)

Download VMRC from Broadcom:
https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware+Remote+Console

Connect using the VMRC URL format:

```
vmrc://[admin-user]@[SIEM-HOST-IP]:443/?id=vm-XXXX
```

Note: VMRC over the internet requires SSL certificate configuration. For LAN use, the
basic setup above is sufficient.

---

### Option C: SSH X11 Forwarding (lightweight, for occasional use)

```bash
ssh -X [admin-user]@[SIEM-HOST-IP]
vmware &
```

This forwards the VMware GUI over X11. Requires X server on the client (Kali has one).
Performance is acceptable on a LAN but laggy over WAN.

---

## Step 7 — Firewall Configuration

If UFW is active on [SIEM-HOST]:

```bash
# Check UFW status
sudo ufw status

# Allow VNC ports (one per VM)
sudo ufw allow 5901:5910/tcp comment 'VMware VNC consoles'

# Allow VMRC (if using remote workstation server feature)
sudo ufw allow 443/tcp comment 'VMware VMRC'
sudo ufw allow 902/tcp comment 'VMware host agent'
```

---

## Step 8 — Creating Your First VM (Headless)

### Option A: Create a VMX manually

```bash
mkdir -p /home/[admin-user]/vmware/ubuntu-test
cat > /home/[admin-user]/vmware/ubuntu-test/ubuntu-test.vmx << 'EOF'
.encoding = "UTF-8"
config.version = "8"
virtualHW.version = "21"
memsize = "2048"
numvcpus = "2"
guestOS = "ubuntu-64"
displayName = "ubuntu-test"
scsi0.present = "TRUE"
scsi0.virtualDev = "pvscsi"
scsi0:0.present = "TRUE"
scsi0:0.fileName = "ubuntu-test.vmdk"
ide1:0.present = "TRUE"
ide1:0.fileName = "/path/to/ubuntu-24.04-live-server-amd64.iso"
ide1:0.deviceType = "cdrom-image"
ethernet0.present = "TRUE"
ethernet0.connectionType = "nat"
ethernet0.virtualDev = "vmxnet3"
usb.present = "TRUE"
RemoteDisplay.vnc.enabled = "TRUE"
RemoteDisplay.vnc.port = "5901"
RemoteDisplay.vnc.password = "[REDACTED]"
EOF
```

Create the disk:

```bash
vmware-vdiskmanager -c -s 40GB -a pvscsi -t 0 \
  /home/[admin-user]/vmware/ubuntu-test/ubuntu-test.vmdk
```

Start it:

```bash
vmrun -T ws start /home/[admin-user]/vmware/ubuntu-test/ubuntu-test.vmx nogui
```

Connect via VNC tunnel to complete OS installation:

```bash
# From Kali:
ssh -L 5901:localhost:5901 [admin-user]@[SIEM-HOST-IP] -N &
vncviewer localhost:5901
```

### Option B: Use vmware-vdiskmanager + ovftool for importing OVA/OVF appliances

```bash
# Import an OVA (adjust paths)
ovftool /path/to/appliance.ova /home/[admin-user]/vmware/imported-vm/
vmrun -T ws start /home/[admin-user]/vmware/imported-vm/imported-vm.vmx nogui
```

---

## Verification Checklist

After full installation, confirm each item:

```bash
# 1. VMware version
vmware --version

# 2. vmrun available
vmrun --version

# 3. Kernel modules loaded
lsmod | grep -E 'vmmon|vmnet'

# 4. Services running
systemctl status vmware vmware-networks

# 5. Virtual network adapters present
ip link show | grep vmnet

# 6. /dev/vmmon device present
ls -la /dev/vmmon

# 7. List any running VMs
vmrun list
```

---

## Rollback / Recovery

### Remove VMware Workstation completely

```bash
sudo vmware-installer --uninstall-product vmware-workstation
# Confirm with 'yes' when prompted

# Remove residual config
sudo rm -rf /etc/vmware /usr/lib/vmware ~/.vmware

# Remove kernel module persistence if added
sudo rm -f /etc/modules-load.d/vmware.conf

# Restore open-vm-tools if removed
sudo apt install -y open-vm-tools
```

### If kernel modules fail to compile after a kernel update

VMware kernel modules (vmmon, vmnet) must be recompiled after kernel upgrades:

```bash
sudo vmware-modconfig --console --install-all
```

Or manually:

```bash
sudo /etc/init.d/vmware stop
cd /usr/lib/vmware/modules/source/
tar xf vmmon.tar
tar xf vmnet.tar
# Patch if needed (see vmware-host-modules GitHub for kernel compatibility patches)
# https://github.com/mkubecek/vmware-host-modules
sudo vmware-modconfig --console --install-all
sudo /etc/init.d/vmware start
```

### If a VM fails to start

```bash
# Check vmware log
cat ~/.vmware/vmware.log | tail -50

# Check kernel ring buffer for module errors
dmesg | grep -i vmmon | tail -20

# Force-kill a stuck VM
vmrun list
vmrun -T ws stop /path/to/vm.vmx hard
```

---

## Notes & Gotchas

1. **Kernel module recompilation after updates:** Every time `apt upgrade` updates the
   kernel, VMware's vmmon/vmnet modules must be recompiled. Consider pinning the kernel
   or automating this with a post-install hook, or use the vmware-host-modules community
   patches for better compatibility:
   https://github.com/mkubecek/vmware-host-modules

2. **open-vm-tools conflict:** The installed `open-vm-tools` package is a *guest agent*
   (meant for when THIS machine is a VM guest). On a bare-metal VMware host it is
   irrelevant but generally harmless. Remove it if you see any conflicts.

3. **RAM constraint:** 7.6 GB total RAM is modest for a hypervisor. Keep VM RAM
   allocations conservative (512MB-1GB for lightweight servers). Swap is available but
   slow — avoid relying on it for VM memory.

4. **VNC password is cleartext in .vmx:** Use SSH tunneling (Step 6 Option A) rather
   than exposing VNC ports directly. Never bind VNC to 0.0.0.0 on an internet-facing
   server.

5. **vmware-workstation-server vs vmrun:** The `vmware-workstation-server` service
   (shared VMs feature) is separate from simply running VMs with `vmrun`. For headless
   server use, `vmrun` alone is sufficient and has fewer attack surfaces.

6. **Broadcom account required:** As of 2024, downloading VMware Workstation Pro requires
   a free Broadcom account. The bundle cannot be obtained without one.

7. **Version compatibility:** VMware Workstation 17.x requires kernel headers. Ubuntu
   24.04 ships with kernel 6.8.x — fully supported as of Workstation 17.5+. If you hit
   module compilation errors, check:
   https://github.com/mkubecek/vmware-host-modules/branches

8. **Disk location for VMs:** Store VM files on the NVMe root partition
   (/home/[admin-user]/vmware/ or /srv/vmware/) for best performance. The 437 GB free is
   more than adequate.

---

## Quick Reference — vmrun Command Cheatsheet

| Action              | Command |
|---------------------|---------|
| Start VM (headless) | `vmrun -T ws start /path/vm.vmx nogui` |
| Stop gracefully     | `vmrun -T ws stop /path/vm.vmx soft` |
| Hard power off      | `vmrun -T ws stop /path/vm.vmx hard` |
| Suspend             | `vmrun -T ws suspend /path/vm.vmx` |
| List running VMs    | `vmrun list` |
| Take snapshot       | `vmrun -T ws snapshot /path/vm.vmx "name"` |
| List snapshots      | `vmrun -T ws listSnapshots /path/vm.vmx` |
| Revert snapshot     | `vmrun -T ws revertToSnapshot /path/vm.vmx "name"` |
| Delete snapshot     | `vmrun -T ws deleteSnapshot /path/vm.vmx "name"` |
| Clone VM            | `vmrun -T ws clone /path/vm.vmx /dest/clone.vmx full` |

---

*Walkthrough generated: 2026-03-20 | System: [SIEM-HOST] ([SIEM-HOST-IP]) | Ubuntu 24.04.4 LTS*
