# Session 04 — Windows AD Lab
**Date:** April 18, 2026
**Machine:** Aperture (10.0.10.20)
**Objective:** Deploy Windows Server 2022 Domain Controller and Windows 11 workstation, configure aperture.science AD domain, wire Wazuh agents into both machines

---

## Table of Contents
1. [Overview](#overview)
2. [Phase 1 — DC01 VM Creation](#phase-1--dc01-vm-creation)
3. [Phase 2 — Windows Server 2022 Install](#phase-2--windows-server-2022-install)
4. [Phase 3 — Domain Controller Promotion](#phase-3--domain-controller-promotion)
5. [Phase 4 — CHELL VM Creation](#phase-4--chell-vm-creation)
6. [Phase 5 — Windows 11 Install](#phase-5--windows-11-install)
7. [Phase 6 — Domain Join CHELL](#phase-6--domain-join-chell)
8. [Phase 7 — AD Users and Groups](#phase-7--ad-users-and-groups)
9. [Phase 8 — Wazuh Agents](#phase-8--wazuh-agents)
10. [Session Status](#session-status)
11. [Lessons Learned](#lessons-learned)

---

## Overview

### Domain
- **Domain Name:** `aperture.science`
- **NetBIOS Name:** `APERTURE`
- **Domain Controller:** DC01 (10.0.10.21)
- **Workstation:** CHELL (DHCP)

### VM Summary
| VM ID | Name | OS | RAM | Role |
|---|---|---|---|---|
| 103 | dc01 | Windows Server 2022 | 4GB | Domain Controller / DNS |
| 104 | chell | Windows 11 Pro | 6GB | Domain-joined workstation |

---

## Phase 1 — DC01 VM Creation

Created VM 103 via Proxmox GUI:

| Setting | Value |
|---|---|
| VM ID | 103 |
| Name | dc01 |
| Machine | q35 |
| BIOS | SeaBIOS |
| SCSI Controller | VirtIO SCSI single |
| CPU | 2 cores, host type |
| RAM | 4096MB |
| Disk | 60GB, VirtIO Block, local-lvm |
| Network | vmbr0, VirtIO |

Added VirtIO drivers ISO as second CD drive (IDE 1).

> **Note:** Use SeaBIOS — OVMF causes boot failures on these VMs (`failed to start boot0002 uefi qemu`).

Download VirtIO drivers ISO to Aperture:
```bash
wget -O /var/lib/vz/template/iso/virtio-win.iso https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

---

## Phase 2 — Windows Server 2022 Install

**ISO:** Windows Server 2022 Evaluation (free 180-day, no license key required)

### Install Steps
1. Boot from Windows Server 2022 ISO
2. Select **Windows Server 2022 Standard Evaluation (Desktop Experience)**
3. Custom install
4. Load VirtIO storage driver at disk selection screen:
   - Browse → `virtio-win → viostor → 2k22 → amd64`
   - Select `viostor.inf`
5. 60GB disk appears — select and install

### Post-Install
```powershell
# Rename computer
Rename-Computer -NewName "DC01" -Restart
```

### Set Static IP
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.10.21 -PrefixLength 24
New-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix 0.0.0.0/0 -NextHop 10.0.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

> **Note:** `-DefaultGateway` is not a valid parameter for `New-NetIPAddress` on Windows Server 2022 — use `New-NetRoute` for the gateway separately.

> **Note:** VirtIO network driver must be installed manually after OS install — Device Manager → Unknown Device → Update Driver → browse to `NetKVM\2k22\amd64` on the VirtIO ISO.

---

## Phase 3 — Domain Controller Promotion

```powershell
# Install AD DS role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promote to domain controller
Install-ADDSForest -DomainName "aperture.science" -DomainNetBiosName "APERTURE" -InstallDns -Force
```

Enter DSRM password when prompted — note it in credentials reference. DC01 reboots automatically after promotion.

### Verify Domain
```powershell
Get-ADDomain
```

Snapshot taken: `domain-controller-baseline`

---

## Phase 4 — CHELL VM Creation

Created VM 104 via Proxmox GUI:

| Setting | Value |
|---|---|
| VM ID | 104 |
| Name | chell |
| Machine | q35 |
| BIOS | SeaBIOS |
| SCSI Controller | VirtIO SCSI single |
| CPU | 2 cores, host type |
| RAM | 6144MB |
| Disk | 60GB, VirtIO Block, local-lvm |
| Network | vmbr0, VirtIO |

Added VirtIO drivers ISO as second CD drive (IDE 1).

---

## Phase 5 — Windows 11 Install

**ISO:** Windows 11 Pro

### Bypass Secure Boot Requirement
Windows 11 installer requires Secure Boot — bypass via registry during setup:

Press **Shift+F10** at the installer screen to open a command prompt:
```cmd
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassTPMCheck /t REG_DWORD /d 1
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassSecureBootCheck /t REG_DWORD /d 1
reg add HKLM\SYSTEM\Setup\LabConfig /v BypassRAMCheck /t REG_DWORD /d 1
```

Click Back then Next to continue past the Secure Boot check.

### Load VirtIO Storage Driver
At disk selection screen browse to VirtIO ISO:
```
virtio-win → amd64 → (vioscsi controller, not passthrough)
```

> **Note:** Windows 11 VirtIO driver may not be under a `w11` folder — look for a standalone `amd64` folder at the root level of the ISO.

### Local Account Setup
- Skip Microsoft account — use local account option
- Username: `chell`

### Post-Install
```powershell
# Rename computer
Rename-Computer -NewName "CHELL" -Restart
```

Install VirtIO network driver via Device Manager → browse to `NetKVM\w11\amd64` on VirtIO ISO.

---

## Phase 6 — Domain Join CHELL

Set DNS to point to DC01 before joining:

**Settings → Network & Internet → Ethernet → DNS → Manual → 10.0.10.21**

Then join the domain:
```powershell
Add-Computer -DomainName "aperture.science" -Credential APERTURE\Administrator -Restart
```

Log in as `APERTURE\Administrator` after reboot.

Snapshot taken: `domain-joined-baseline`

---

## Phase 7 — AD Users and Groups

Run on DC01 in PowerShell as Administrator:

```powershell
# Standard user — initial foothold target
New-ADUser -Name "Doug Rattmann" -SamAccountName "doug.rattmann" -UserPrincipalName "doug.rattmann@aperture.science" -AccountPassword (ConvertTo-SecureString "<LAB_PASSWORD>" -AsPlainText -Force) -Enabled $true

# Local admin — privilege escalation path
New-ADUser -Name "Caroline" -SamAccountName "caroline" -UserPrincipalName "caroline@aperture.science" -AccountPassword (ConvertTo-SecureString "<LAB_PASSWORD>" -AsPlainText -Force) -Enabled $true

# Domain admin — high value target
New-ADUser -Name "Cave Johnson" -SamAccountName "cave.johnson" -UserPrincipalName "cave.johnson@aperture.science" -AccountPassword (ConvertTo-SecureString "<LAB_PASSWORD>" -AsPlainText -Force) -Enabled $true

# Service account — Kerberoasting target
New-ADUser -Name "svc-backup" -SamAccountName "svc-backup" -UserPrincipalName "svc-backup@aperture.science" -AccountPassword (ConvertTo-SecureString "<SVC_PASSWORD>" -AsPlainText -Force) -Enabled $true -PasswordNeverExpires $true

# Assign group memberships
Add-ADGroupMember -Identity "Domain Admins" -Members "cave.johnson"
Add-ADGroupMember -Identity "Administrators" -Members "caroline"

# Register SPN on service account for Kerberoasting
Set-ADUser -Identity "svc-backup" -ServicePrincipalNames @{Add="HTTP/dc01.aperture.science"}
```

> **Note:** Use intentionally weak passwords for lab accounts to enable password attack exercises (Kerberoasting, AS-REP Roasting). Passwords are stored in your local credentials reference — not committed to the repo.

### AD User Reference
| Account | Role | Attack Target |
|---|---|---|
| cave.johnson | Domain Admin | High-value credential target |
| caroline | Local Admin | Privilege escalation path |
| doug.rattmann | Standard User | Initial foothold |
| svc-backup | Service Account | Kerberoasting |

Snapshot taken: `ad-users-baseline`

---

## Phase 8 — Wazuh Agents

Agents deployed via Wazuh dashboard deploy wizard (`https://10.0.10.10` → Agents → Deploy new agent → Windows).

| Machine | Agent Name | Status |
|---|---|---|
| DC01 | dc01 | Active |
| CHELL | chell | Active |

> **Note:** GLaDOS does not appear as a Wazuh agent — it is the Wazuh manager and monitors itself natively. This is expected.

---

## Session Status

### Aperture (10.0.10.20)
| VM | Status |
|---|---|
| Kali Linux (ID 100) | Off |
| Metasploitable (ID 101) | Off |
| REMnux (ID 102) | Off |
| DC01 (ID 103) | Running — aperture.science domain controller |
| CHELL (ID 104) | Running — domain joined |

### Wazuh Active Agents
| Agent | Status |
|---|---|
| aperture | Active |
| dc01 | Active |
| chell | Active |

### AD Lab Ready For
- Kerberoasting
- AS-REP Roasting
- BloodHound enumeration
- Pass-the-Hash
- Mimikatz credential dumping

---

## Lessons Learned

- **OVMF causes boot failures on these VMs** — use SeaBIOS for all Windows VMs on Aperture
- **`-DefaultGateway` is not a valid parameter for `New-NetIPAddress`** — use `New-NetRoute` separately for the default gateway
- **VirtIO network driver is not installed automatically** — must install manually via Device Manager after OS install
- **Windows 11 requires Secure Boot bypass in a lab** — use LabConfig registry keys via Shift+F10 during setup
- **Windows 11 VirtIO storage driver may be in a standalone `amd64` folder** — not always under a named OS subfolder on the ISO
- **CHELL DNS must point to DC01 before domain join** — TZ370 DNS will not resolve `aperture.science`
- **GLaDOS is the Wazuh manager, not an agent** — it won't appear in the agents list, this is expected

---

*Session completed: April 18, 2026*
