# Home Lab Build — Session Notes
**Date:** March 30, 2026  
**Author:** Watcher  
**Session:** Initial build, firewall config, and GLaDOS SOC machine setup

---

## Table of Contents
1. [Hardware Summary](#hardware-summary)
2. [Network Architecture](#network-architecture)
3. [SonicWall TZ370 Configuration](#sonicwall-tz370-configuration)
4. [SSL-VPN Setup](#ssl-vpn-setup)
5. [GLaDOS — Ubuntu Server Install](#glados--ubuntu-server-install)
6. [Wazuh Installation](#wazuh-installation)
7. [Suricata Installation](#suricata-installation)
8. [Zeek Installation](#zeek-installation)
9. [UFW Firewall Rules](#ufw-firewall-rules)
10. [Current Status](#current-status)
11. [Next Steps](#next-steps)

---

## Hardware Summary

### Lab Machines
| Hostname | Hardware | RAM | Storage | Role | IP |
|---|---|---|---|---|---|
| glados | ThinkCentre M70Q (i7-10700T) | 32GB DDR4 | 1TB SSD (Seagate FireCuda 520) | SOC / SIEM | 10.0.10.10 |
| aperture | ThinkCentre M70Q (i7-10700T) | 16GB DDR4 | TBD | Proxmox / CTF | 10.0.10.20 |
| TBD | Raspberry Pi 5 (8GB) | 8GB | TBD | Home Assistant OS | 10.0.10.30 |
| TBD | Raspberry Pi 4 | TBD | TBD | Pi-hole / Uptime Kuma / Node-RED | 10.0.10.40 |

### Network Hardware
| Device | Role |
|---|---|
| SonicWall TZ370 Gen 7 | Lab firewall / gateway |
| ASUS RT-BE88U | Home network router |
| RackMate TT | Mini rack (vertical mode) |

### Main PC
- OS: Windows 11
- CPU: AMD Ryzen 7 7800X3D
- RAM: 32GB
- GPU: RTX 4070

---

## Network Architecture

```
Internet
    ↓
AT&T Fiber → ONT → AT&T Gateway
    ↓
RT-BE88U (home network — <HOME_SUBNET_RANGE>)
    ↓
TZ370 WAN (<WAN_IP>)
    ↓
TZ370 LAN (10.0.10.1)
    ↓
├── X0 → GLaDOS        10.0.10.10  (SOC/SIEM)
├── X2 → Aperture      10.0.10.20  (Proxmox/CTF)
├── X3 → Pi 5          10.0.10.30  (Home Assistant)
└── X4 → Pi 4          10.0.10.40  (Pi-hole/Uptime Kuma)
```

### Subnet Information
| Network | Subnet | Purpose |
|---|---|---|
| Home Network | <HOME_SUBNET> | Personal devices, RT-BE88U |
| Lab Network | 10.0.10.0/24 | All lab devices |
| DHCP Pool | 10.0.10.50–200 | Dynamic assignments |
| Static Range | 10.0.10.2–49 | Reserved for lab machines |
| VPN Pool | 10.10.10.0/24 | SSL-VPN clients (separate subnet from lab) |

---

## SonicWall TZ370 Configuration

### Access
- Management IP (during setup): `https://192.168.168.168:4433`
- Management IP (from home network): `https://<WAN_IP>:4433`
- Management IP (from lab): `https://10.0.10.1`
- Default login: `admin` / `password` (changed on first login)

### Factory Reset Procedure
1. Power on TZ370
2. Hold reset pinhole button for 10-15 seconds
3. Wait for LED to flash, release
4. Wait 2-3 minutes for reboot
5. Access at `https://192.168.168.168:4433`

### Interface Configuration
| Port | Mode | Zone | IP | Connected To |
|---|---|---|---|---|
| X0 | Static IP | LAN | 10.0.10.1 | GLaDOS (SIEM) |
| X1 | Static IP | WAN | <WAN_IP> | RT-BE88U LAN port |
| X2 | PortShield → X0 | LAN | — | Aperture (Proxmox) |
| X3 | PortShield → X0 | LAN | — | Pi 5 |
| X4 | PortShield → X0 | LAN | — | Pi 4 |
| MGMT | — | — | 192.168.168.168 | Setup only |

> **Note:** X2, X3, X4 set to PortShield Switch Mode pointing to X0. This groups them all under the same LAN zone and subnet without needing individual IPs.

### DHCP Configuration
- **Enabled on:** X0 (LAN)
- **Range Start:** 10.0.10.50
- **Range End:** 10.0.10.200
- **Default Gateway:** 10.0.10.1
- **DNS Server 1:** 8.8.8.8
- **DNS Server 2:** 8.8.4.4
- **Lease Time:** 1440 minutes

### Address Objects Created
| Name | Zone | Type | Value |
|---|---|---|---|
| Lab-Network | LAN | Network | 10.0.10.0/24 |
| Home-Network | WAN | Network | <HOME_SUBNET> |
| SSLVPN-Pool | SSLVPN | Network | 10.10.10.0/24 |

### Firewall Rules
| Name | Action | From | To | Purpose |
|---|---|---|---|---|
| Block-Lab-to-Home | Deny | LAN (Lab-Network) | WAN (Home-Network) | Prevents lab devices reaching home network |
| SSLVPN-to-LAN | Allow | SSLVPN | LAN | Allows VPN clients to reach lab |

---

## SSL-VPN Setup

### Server Settings
- **SSL VPN Port:** 4433
- **Certificate:** Self-signed
- **Authentication:** Password
- **User Domain:** LocalDomain
- **WAN Zone:** Enabled

### Device Profile (Default)
- **VPN IP Pool:** 10.10.10.0/24 (separate subnet from lab — best practice)
- **Why separate subnet:** Keeps VPN clients isolated from lab devices, full /24 gives 254 IPs, TZ370 auto-routes between SSLVPN zone and LAN zone
- **DNS Server 1:** 8.8.8.8
- **DNS Server 2:** 8.8.4.4
- **Client Route:** 10.0.10.0/24 (split tunnel — only lab traffic goes through VPN)

### VPN User Account
- **Username:** watcher
- **Group:** SSLVPN Services
- **VPN Access:** Lab-Network (10.0.10.0/24)

### Connecting from Windows PC
1. Install SonicWall NetExtender on Windows
2. Open NetExtender
3. Server: `<WAN_IP>:4433`
4. Username: `watcher`
5. Password: VPN password
6. Domain: `LocalDomain`
7. Click Connect

### After VPN Connected
- SSH to GLaDOS: `ssh watcher@10.0.10.10`
- Access Wazuh: `https://10.0.10.10`
- Access TZ370: `https://10.0.10.1`

---

## GLaDOS — Ubuntu Server Install

### System Info
- **Hostname:** glados
- **OS:** Ubuntu Server 24.04.4 LTS
- **IP:** 10.0.10.10/24
- **Gateway:** 10.0.10.1
- **Interface:** eno2
- **MAC:** 84:a9:38:7f:2e:50
- **Username:** watcher

### Installation Steps
1. Downloaded Ubuntu Server 24.04 LTS ISO
2. Flashed to USB using Rufus (GPT partition scheme, UEFI)
3. Booted M70Q from USB (F12 for boot menu)
4. Selected language and keyboard
5. Network: configured eno2 as static IP 10.0.10.10/24, gateway 10.0.10.1
6. Proxy: left blank
7. Storage: Use entire disk, Seagate FireCuda 520 1TB, LVM enabled, no LUKS
8. Profile: name=Daniel, server=glados, username=watcher
9. Ubuntu Pro: skipped
10. OpenSSH: enabled with password authentication
11. Snaps: none selected
12. Install completed, USB removed, rebooted

### Post-Install Commands
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install curl wget git net-tools htop ufw -y

# Configure UFW
sudo ufw allow 22/tcp
sudo ufw enable
```

### Verify Network
```bash
ip a                    # Confirm 10.0.10.10 on eno2
ping -c 4 8.8.8.8      # Confirm internet access
```

### SSH Access from Windows PC (via VPN)
```powershell
ssh watcher@10.0.10.10
```

---

## Wazuh Installation

### Version Installed
- Wazuh 4.14.2 (latest as of March 2026)
- All-in-one installation (Manager + Indexer + Dashboard)

### Install Commands
```bash
# Download installer
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh

# Run all-in-one install
sudo bash wazuh-install.sh -a
```

> **Important:** Save the admin password displayed at the end of installation immediately. It cannot be easily recovered.

### UFW Rules Added for Wazuh
```bash
sudo ufw allow 443/tcp      # Dashboard
sudo ufw allow 1514/tcp     # Agent communication
sudo ufw allow 1515/tcp     # Agent registration
sudo ufw allow 55000/tcp    # Wazuh API
sudo ufw reload
```

### Access Dashboard
- URL: `https://10.0.10.10`
- Username: `admin`
- Password: (saved from install output)

### Verify Services
```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

---

## Suricata Installation

### Version
- Suricata (latest stable from OISF PPA)

### Install Commands
```bash
# Add Suricata PPA
sudo add-apt-repository ppa:oisf/suricata-stable -y

# Install Suricata
sudo apt install suricata -y

# Update threat signatures
sudo suricata-update
```

### Configuration
```bash
# Edit config
sudo nano /etc/suricata/suricata.yaml
```

Changes made:
- `af-packet interface:` changed from `eth0` to `eno2`
- `HOME_NET:` changed to `[10.0.10.0/24]`

Used sed for automated replacement:
```bash
sudo sed -i 's/interface: eth0/interface: eno2/g' /etc/suricata/suricata.yaml
```

### Start and Enable
```bash
sudo systemctl enable suricata
sudo systemctl start suricata
sudo systemctl status suricata
```

### Test Suricata is Working
```bash
# Generate test alert
curl http://testmynids.org/uid/index.html

# Check for alert
sudo tail -f /var/log/suricata/fast.log
```

### Connect Suricata to Wazuh
```bash
sudo tee -a /var/ossec/etc/ossec.conf << 'EOF'
<ossec_config>
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>
</ossec_config>
EOF

sudo systemctl restart wazuh-manager
```

---

## Zeek Installation

### Version
- Zeek 8.0 (latest LTS as of March 2026)

### Add Repository
```bash
# Add Zeek repo for Ubuntu 24.04
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list

# Add GPG key
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null

# Update and install
sudo apt update
sudo apt install zeek-8.0 -y
```

### Add Zeek to PATH
```bash
echo "export PATH=$PATH:/opt/zeek/bin" >> ~/.bashrc
source ~/.bashrc
```

### Fix sudo PATH for Zeek
```bash
sudo visudo
# Add :/opt/zeek/bin to the secure_path line
```

### Configure Interface
```bash
sudo nano /opt/zeek/etc/node.cfg
# Change interface=eth0 to interface=eno2
```

### Configure Network
```bash
sudo nano /opt/zeek/etc/networks.cfg
# Add: 10.0.10.0/24    Lab Network
```

### Deploy Zeek
```bash
sudo /opt/zeek/bin/zeekctl deploy
```

### Verify Running
```bash
sudo /opt/zeek/bin/zeekctl status
# Should show: zeek standalone localhost running
```

### Auto Start on Boot
```bash
sudo crontab -e
# Add line: @reboot /opt/zeek/bin/zeekctl start
```

### Connect Zeek to Wazuh
```bash
sudo tee -a /var/ossec/etc/ossec.conf << 'EOF'
<ossec_config>
  <localfile>
    <log_format>json</log_format>
    <location>/opt/zeek/logs/current/conn.log</location>
  </localfile>
</ossec_config>
EOF

sudo systemctl restart wazuh-manager
```

### Log Locations
```
/opt/zeek/logs/current/conn.log     # Network connections
/opt/zeek/logs/current/dns.log      # DNS queries
/opt/zeek/logs/current/http.log     # HTTP traffic
/opt/zeek/logs/current/ssl.log      # SSL/TLS traffic
```

---

## UFW Firewall Rules

### Complete UFW Rule Set on GLaDOS
```bash
sudo ufw allow 22/tcp       # SSH
sudo ufw allow 443/tcp      # Wazuh Dashboard (HTTPS)
sudo ufw allow 1514/tcp     # Wazuh agent communication
sudo ufw allow 1515/tcp     # Wazuh agent registration
sudo ufw allow 55000/tcp    # Wazuh API
sudo ufw enable
sudo ufw reload
```

### Check Status
```bash
sudo ufw status
```

> **Lesson learned:** UFW was blocking port 443 which prevented the Wazuh dashboard from being reachable externally. Always check UFW when a service is running but not reachable.

---

## Current Status

### GLaDOS (10.0.10.10) ✅
| Service | Status |
|---|---|
| Ubuntu Server 24.04 | Running |
| Wazuh Manager | Running |
| Wazuh Indexer | Running |
| Wazuh Dashboard | Running — https://10.0.10.10 |
| Suricata | Running — feeding into Wazuh |
| Zeek 8.0 | Running — feeding into Wazuh |
| UFW | Active — all ports configured |
| SSH | Accessible via VPN |

### TZ370 ✅
| Feature | Status |
|---|---|
| WAN | Connected — <WAN_IP> |
| LAN | Configured — 10.0.10.1 |
| DHCP | Active — 10.0.10.50-200 |
| PortShield (X2-X4) | Configured |
| Block lab-to-home rule | Active |
| SSL-VPN | Active — watcher user created |

### SSL-VPN ✅
- NetExtender installed on Windows PC
- VPN connects successfully to <WAN_IP>:4433
- SSH to GLaDOS working over VPN

---

## Next Steps

### Session 2 — Aperture (Proxmox Machine) ✅
- [x] Install Proxmox VE on second M70Q (500GB SSD)
- [x] Configure network: static IP 10.0.10.20
- [x] Disable enterprise repos, add no-subscription repo (trixie)
- [x] System fully updated
- [x] Static DHCP binding added in TZ370 — 54:05:db:cb:47:76 → 10.0.10.20
- [x] GLaDOS static DHCP binding also added — 84:a9:38:7f:2e:50 → 10.0.10.10
- [x] Kali Linux VM (ID 100) — installed, clean-install snapshot taken
- [x] Metasploitable VM (ID 101) — imported VMDK, booted successfully
- [x] Wazuh agent installed on Aperture via APT repo — pointing to 10.0.10.10
- [x] Wazuh agent installed on Kali (agent 002) — pointing to 10.0.10.10
- [x] Suricata installed on Aperture monitoring vmbr0
- [x] Suricata on Aperture connected to Wazuh agent
- [x] All 3 agents verified in Wazuh dashboard — glados, aperture, kali

### Session 2 — First Attack Exercise ✅
- [x] nmap -sV -O scan against Metasploitable (10.0.10.106)
- [x] Suricata detected Kali OS fingerprint from DHCP request
- [x] Exploited vsftpd 2.3.4 backdoor via Metasploit — got root shell
- [x] MITRE ATT&CK detections fired — Remote Services, Valid Accounts, Password Guessing
- [x] 445 total alerts generated across all agents
- [x] Full attack/detect cycle completed

### Wazuh Agent Notes
- Wazuh install script blocked by TZ370 DPI-SSL — use APT repo method instead:
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH -o /tmp/wazuh.key
cat /tmp/wazuh.key | sudo gpg --dearmor | sudo tee /usr/share/keyrings/wazuh.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
sudo WAZUH_MANAGER='10.0.10.10' apt install wazuh-agent -y
sudo systemctl enable wazuh-agent && sudo systemctl start wazuh-agent
```

### Suricata on Aperture (vmbr0)
- Monitors all VM-to-VM traffic inside Proxmox
- Interface: vmbr0
- Logs: /var/log/suricata/eve.json
- Connected to Wazuh agent on Aperture

### Metasploitable VM Details
- VM ID: 101
- IP: 10.0.10.106 (DHCP)
- Login: msfadmin / msfadmin
- Key vulnerabilities found by nmap:
  - Port 21: vsftpd 2.3.4 (backdoored)
  - Port 22: OpenSSH 4.7p1
  - Port 23: Telnet
  - Port 1524: Bindshell — root shell
  - Port 3306: MySQL 5.0.51a
  - Port 5432: PostgreSQL 8.3
  - Port 6667: UnrealIRCd

### Session 3 — Wazuh Agents
- [x] Install Wazuh agent on Aperture ✅
- [x] Install Wazuh agent on Kali ✅
- [ ] Install Wazuh agent on Pi 5
- [ ] Install Wazuh agent on Pi 4
- [ ] Verify all agents appear in Wazuh dashboard

### Session 4 — Additional SOC Tools
- [ ] Install Graylog on GLaDOS
- [ ] Install TheHive + Cortex on GLaDOS
- [ ] Configure Cortex analyzers (VirusTotal, Shodan)
- [ ] Install Arkime for full packet capture

### Session 5 — Pi Setup
- [ ] Install Home Assistant OS on Pi 5
- [ ] Install Pi-hole on Pi 4
- [ ] Install Uptime Kuma on Pi 4
- [ ] Install Node-RED on Pi 4

---

## Key Commands Quick Reference

```bash
# SSH into GLaDOS (from Windows via VPN)
ssh watcher@10.0.10.10

# Check all SOC services
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo systemctl status suricata
sudo /opt/zeek/bin/zeekctl status

# Check resource usage
free -h       # RAM
df -h         # Disk
htop          # CPU and processes

# Check Suricata alerts
sudo tail -f /var/log/suricata/fast.log
sudo tail -f /var/log/suricata/eve.json

# Check Zeek logs
ls /opt/zeek/logs/current/
sudo tail -f /opt/zeek/logs/current/conn.log

# Restart all SOC services
sudo systemctl restart wazuh-manager
sudo systemctl restart suricata
sudo /opt/zeek/bin/zeekctl restart

# UFW status
sudo ufw status

# Network check
ip a
ping -c 4 8.8.8.8
```

---

## Credentials Reference
> Store these securely — do not commit to public git repo

| Service | Username | Password Location |
|---|---|---|
| TZ370 GUI | admin | Password manager |
| Wazuh Dashboard | admin | Password manager |
| SSL-VPN | watcher | Password manager |
| GLaDOS SSH | watcher | Password manager |
| Proxmox (future) | root | Password manager |

---

## Notes and Lessons Learned

- **UFW blocks everything by default** — always add UFW rules after installing services that need to be reachable externally
- **Zeek 8.0 installs to /opt/zeek/bin** — not in default PATH, needs manual export or visudo fix for sudo commands
- **SonicWall default IP** is `192.168.168.168:4433` not `192.168.168.1` on Gen 7
- **PortShield Switch Mode** is the correct way to add LAN ports on TZ370 without assigning individual IPs
- **SSL-VPN needs two firewall rules** — WAN enabled on server settings AND an access rule SSLVPN→LAN
- **VPN pool should be on a separate subnet** — using 10.10.10.0/24 for VPN clients instead of a range inside 10.0.10.0/24 is best practice. Keeps VPN clients isolated, gives more IPs, and TZ370 auto-routes between SSLVPN and LAN zones so main PC can reach lab regardless of home network subnet
- **Postfix is a Zeek dependency** — select "Local only" during installation, not needed for homelab
- **RAM crisis (2026)** — DDR4 and DDR5 prices have increased 100-500% due to AI demand. Only buy machines with RAM already installed, don't plan on upgrading separately
- **Double NAT is intentional** — TZ370 behind RT-BE88U provides lab isolation from home network

---

*Session completed: March 30, 2026*  
*Next session: Proxmox setup on Aperture (second M70Q)*
