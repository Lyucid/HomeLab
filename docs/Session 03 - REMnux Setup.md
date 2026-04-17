# Session 03 — REMnux Setup on Aperture
**Date:** April 17, 2026
**Machine:** Aperture (10.0.10.20)
**Objective:** Upgrade Aperture RAM to 32GB, deploy REMnux VM for malware analysis

---

## Table of Contents
1. [Hardware Upgrade](#hardware-upgrade)
2. [Phase 1 — Create REMnux VM](#phase-1--create-remnux-vm)
3. [Phase 2 — Import REMnux OVA](#phase-2--import-remnux-ova)
4. [Phase 3 — Attach Disk and Configure Boot](#phase-3--attach-disk-and-configure-boot)
5. [Phase 4 — Network Configuration](#phase-4--network-configuration)
6. [Phase 5 — REMnux Upgrade](#phase-5--remnux-upgrade)
7. [Session Status](#session-status)
8. [Lessons Learned](#lessons-learned)

---

## Hardware Upgrade

Aperture upgraded from 16GB to 32GB DDR4.

| Component | Before | After |
|---|---|---|
| RAM | 16GB DDR4 | 32GB DDR4 |

> **Note:** RAM runs at 2933 MHz despite being rated for 3200 MHz. This is the i7-10700T's platform max — not a configuration issue.

---

## Phase 1 — Create REMnux VM

Created VM 102 via Proxmox GUI with the following settings:

| Setting | Value |
|---|---|
| VM ID | 102 |
| Name | remnux |
| Machine | q35 |
| BIOS | Default (SeaBIOS) |
| SCSI Controller | VirtIO SCSI single |
| CPU | 2 cores, host type |
| RAM | 4096MB |
| Network | vmbr0, VirtIO |
| Disk | None — imported from OVA |

> **Note:** Delete the default disk Proxmox adds during VM creation — the disk comes from the OVA import.

---

## Phase 2 — Import REMnux OVA

Downloaded REMnux OVA from https://docs.remnux.org/install-distro/get-virtual-appliance

Transferred OVA from Windows PC to Aperture:
```powershell
scp "remnux.ova" root@10.0.10.20:/root/
```

Extracted OVA on Aperture:
```bash
cd /root
tar xf remnux.ova
ls *.vmdk.gz
```

> **Note:** REMnux OVA contains a `.vmdk.gz` (gzip compressed) — not a plain VMDK. Must decompress before importing.

Decompress the VMDK:
```bash
gunzip *disk1.vmdk.gz
```

Import disk into VM 102:
```bash
qm importdisk 102 *disk1.vmdk local-lvm
```

---

## Phase 3 — Attach Disk and Configure Boot

After import the disk sits as `unused0` — Proxmox does not attach it automatically.

**Problem:** VM was looping with nothing to boot. `qm config 102` showed:
```
boot: order=ide2
unused0: local-lvm:vm-102-disk-0
```

**Fix — attach disk and set boot order via CLI:**
```bash
qm set 102 --virtio0 local-lvm:vm-102-disk-0
qm set 102 --boot order=virtio0
```

Stop and start the VM — boots into REMnux.

**Default credentials:**
- Username: `remnux`
- Password: `malware`

Change password on first login:
```bash
passwd
```

---

## Phase 4 — Network Configuration

REMnux booted with no network connectivity. The VirtIO interface `enp6s18` was down.

**Temporary fix (one-time):**
```bash
sudo ip link set enp6s18 up
sudo dhcpcd enp6s18
```

**Permanent fix — update netplan config:**

The default netplan config referenced `ens33` (VMware default) instead of `enp6s18`.

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

Updated content:
```yaml
network:
  version: 2
  ethernets:
    enp6s18:
      dhcp4: true
```

Apply:
```bash
sudo netplan apply
```

Network now comes up automatically on boot. REMnux picks up a 10.0.10.x address from TZ370 DHCP.

---

## Phase 5 — REMnux Upgrade

```bash
remnux upgrade
```

> **Note:** First run may fail with `Fatal 5339 salt-call completed but had failed states`. This is a known REMnux issue — reboot and run `remnux upgrade` again. May require two runs to fully complete.

---

## Session Status

### Aperture (10.0.10.20)
| Service | Status |
|---|---|
| Proxmox VE | Running — https://10.0.10.20:8006 |
| Kali Linux VM (ID 100) | Running — clean-install snapshot taken |
| Metasploitable VM (ID 101) | Running — IP 10.0.10.106 |
| REMnux VM (ID 102) | Running — clean-install snapshot taken |
| Wazuh Agent | Running |
| Suricata (vmbr0) | Running |

---

## Lessons Learned

- **REMnux OVA contains a gzip-compressed VMDK** — run `gunzip` before importing; `qm importdisk` cannot handle `.vmdk.gz` directly
- **Proxmox does not auto-attach imported disks** — always verify with `qm config <vmid>` after import; disk will show as `unused0` until manually attached
- **REMnux netplan references ens33 by default** — VMware interface name, not correct for Proxmox VirtIO; update to `enp6s18`
- **remnux upgrade may fail on first run** — Salt state failures are common on first upgrade; reboot and rerun to complete
- **i7-10700T RAM runs at 2933 MHz max** — platform limitation, not configurable in ThinkCentre BIOS

---

*Session completed: April 17, 2026*
