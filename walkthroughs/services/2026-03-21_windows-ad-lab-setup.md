# Windows AD Lab VM Setup (WinServer2022 + Win11)
**Date:** 2026-03-21
**Category:** services
**System:** Ubuntu 24.04 — [SIEM-HOST] ([SIEM-HOST-IP]), VMware Workstation Pro 17.6.3
**Risk Level:** Low

## Overview
Decommissioned the existing Kali Linux VM and provisioned two new VMs for a Windows Active Directory lab:
- **WinServer2022** — future AD Domain Controller (DC), 4 GB RAM, 2 vCPUs, 80 GB disk, VNC port 5910
- **Win11** — future domain-joined workstation, 4 GB RAM, 2 vCPUs, 80 GB disk, VNC port 5911

Both VMs are on VMware NAT (vmnet8) so they share a private network and can reach each other and the internet via the host's NAT. Both are ready to boot once ISOs are uploaded to `/home/[admin-user]/ISOs/`.

## Prerequisites
- VMware Workstation Pro 17.6.3 installed and licensed on [SIEM-HOST]
- `vmrun` and `vmware-vdiskmanager` in PATH
- SSH access as `[admin-user]@[SIEM-HOST-IP]`
- Sufficient disk space (406 GB free on /dev/nvme0n1p2)
- ISOs to upload later:
  - `/home/[admin-user]/ISOs/WinServer2022.iso`
  - `/home/[admin-user]/ISOs/Win11.iso`

## Pre-Change State / Backup
The Kali Linux VM at `~/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/` was stopped and deleted. A compressed archive `~/VMs/kali-linux-2025.4-vmware-amd64.7z` was already present and left untouched — it serves as an implicit backup of the Kali VM if needed.

```bash
# State before removal
vmrun -T ws list
# Confirmed Kali VM was not running at time of deletion
```

## Step-by-Step Instructions

### Step 1 — Remove Kali VM
```bash
vmrun -T ws stop ~/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx soft 2>/dev/null || true
vmrun -T ws deleteVM ~/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx 2>/dev/null || true
rm -rf ~/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/
```
The `|| true` guards prevent script abort if the VM is already stopped or unregistered.

### Step 2 — Create directory structure
```bash
mkdir -p ~/ISOs
mkdir -p ~/VMs/WinServer2022
mkdir -p ~/VMs/Win11
```

### Step 3 — Write WinServer2022.vmx
Path: `/home/[admin-user]/VMs/WinServer2022/WinServer2022.vmx`

Key settings:
| Parameter | Value | Reason |
|-----------|-------|--------|
| guestOS | windows2019srvnext-64 | Best VMware match for Server 2022 |
| virtualHW.version | 21 | WS Pro 17 hardware level |
| memsize | 4096 | 4 GB — minimum comfortable for a DC |
| numvcpus | 2 | Adequate for AD + DNS + DHCP roles |
| firmware | efi | Modern boot; required for Server 2022 |
| ethernet0.connectionType | nat | vmnet8 — shared NAT with Win11 |
| ethernet0.virtualDev | e1000e | Best driver compatibility |
| RemoteDisplay.vnc.port | 5910 | VNC access during OS install |
| RemoteDisplay.vnc.password | [REDACTED] | Lab-only credential |
| sound.present | FALSE | Headless server — no audio needed |

### Step 4 — Write Win11.vmx
Path: `/home/[admin-user]/VMs/Win11/Win11.vmx`

Additional Win11-specific settings beyond the Server 2022 config:
| Parameter | Value | Reason |
|-----------|-------|--------|
| guestOS | windows11-64 | Correct VMware guest type |
| uefi.secureBoot.enabled | TRUE | Win11 hardware requirement |
| tpm.present | TRUE | Win11 hardware requirement |
| tpm.allowListOnly | FALSE | Allow VMware software TPM |
| usb_xhci.present | TRUE | USB 3.0 controller for installer |
| RemoteDisplay.vnc.port | 5911 | Separate VNC port from Server VM |

### Step 5 — Create VMDK disks
`-t 0` = monolithic sparse (thin provisioned — grows as data is written).
`-a lsilogic` = LSI Logic SCSI adapter, matches the VMX scsi0.virtualDev setting.

```bash
vmware-vdiskmanager -c -t 0 -s 80GB -a lsilogic /home/[admin-user]/VMs/WinServer2022/WinServer2022.vmdk
vmware-vdiskmanager -c -t 0 -s 80GB -a lsilogic /home/[admin-user]/VMs/Win11/Win11.vmdk
```

Both reported: `Virtual disk creation successful.`
Actual on-disk size at creation is ~11 MB each (sparse); will grow as Windows installs.

## Verification

```bash
# Disk space — confirmed 406 GB free
df -h /home
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/nvme0n1p2  468G   39G  406G   9% /

# Directory contents
ls -lh ~/VMs/WinServer2022/
# -rw------- 1 [admin-user] [admin-user]  11M WinServer2022.vmdk
# -rw-rw-r-- 1 [admin-user] [admin-user] 1.5K WinServer2022.vmx

ls -lh ~/VMs/Win11/
# -rw------- 1 [admin-user] [admin-user]  11M Win11.vmdk
# -rw-rw-r-- 1 [admin-user] [admin-user] 1.7K Win11.vmx
```

## Next Steps — ISO Upload and Boot

1. Transfer ISOs to the server:
```bash
# From a machine that has the ISOs:
scp WinServer2022.iso [admin-user]@[SIEM-HOST-IP]:~/ISOs/
scp Win11.iso [admin-user]@[SIEM-HOST-IP]:~/ISOs/
```

2. Start a VM:
```bash
vmrun -T ws start /home/[admin-user]/VMs/WinServer2022/WinServer2022.vmx nogui
vmrun -T ws start /home/[admin-user]/VMs/Win11/Win11.vmx nogui
```

3. Connect via VNC to complete OS installation:
   - WinServer2022: `[SIEM-HOST-IP]:5910` password `[REDACTED]`
   - Win11: `[SIEM-HOST-IP]:5911` password `[REDACTED]`

4. Post-install AD setup on WinServer2022:
   - Install AD DS, DNS, DHCP roles via Server Manager or PowerShell
   - Promote to Domain Controller: `Install-ADDSForest -DomainName "lab.local"`
   - Configure DHCP scope on vmnet8 subnet (default: 192.168.136.0/24)

5. Join Win11 to the domain after DC is configured.

## Rollback / Recovery

### To restore the Kali VM
The original VM archive is still present:
```bash
ls -lh ~/VMs/kali-linux-2025.4-vmware-amd64.7z
# Extract:
cd ~/VMs && 7z x kali-linux-2025.4-vmware-amd64.7z
vmrun -T ws start ~/VMs/kali-linux-2025.4-vmware-amd64.vmwarevm/kali-linux-2025.4-vmware-amd64.vmx
```

### To remove the new VMs
```bash
vmrun -T ws stop /home/[admin-user]/VMs/WinServer2022/WinServer2022.vmx hard 2>/dev/null || true
vmrun -T ws stop /home/[admin-user]/VMs/Win11/Win11.vmx hard 2>/dev/null || true
rm -rf /home/[admin-user]/VMs/WinServer2022/
rm -rf /home/[admin-user]/VMs/Win11/
rm -rf /home/[admin-user]/ISOs/   # only if ISOs are not needed
```

## Notes & Gotchas

- **Win11 TPM**: VMware provides a software TPM when `tpm.present = TRUE` and `tpm.allowListOnly = FALSE`. VMware stores the TPM state in a `.nvram` file alongside the VMX. Do not delete it after installation or Win11 will fail to boot (BitLocker/TPM binding).
- **Win11 Secure Boot**: `uefi.secureBoot.enabled = TRUE` is required. If the installer complains about Secure Boot being off, verify this line is present and the VM is not using legacy BIOS mode.
- **Win11 bypass for hardware checks**: If using an evaluation ISO that enforces hardware checks, a `autounattend.xml` with registry bypass keys may be needed, or use the Rufus bypass method on the ISO before uploading.
- **VNC password limit**: VMware VNC passwords are truncated at 8 characters. "[REDACTED]" (7 chars) is fine.
- **vmnet8 DHCP range**: By default vmnet8 uses 192.168.136.0/24 with DHCP. Both VMs will receive IPs in this range automatically. Verify with `vmware-netcfg` or check `/etc/vmware/vmnet8/dhcpd/dhcpd.conf` on the host.
- **SCSI vs NVMe**: LSI Logic SCSI (`lsilogic`) was chosen for broadest Windows driver compatibility without needing to inject NVMe drivers during setup. Can be upgraded post-install.
- **Hardware version 21**: Matches VMware Workstation Pro 17. Do not open these VMX files with older VMware versions without first lowering `virtualHW.version`.
- **VNC security**: VNC with a static password is acceptable for a lab on a trusted LAN. Do not expose port 5910/5911 to the internet without a VPN or SSH tunnel.
