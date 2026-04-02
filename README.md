# Home Lab — SOC & CTF Build

A two-machine home cybersecurity lab built for SOC analyst training, threat detection, and CTF practice.

## Architecture

```
Internet → ISP Gateway → Home Router (<HOME_SUBNET>/24)
                              ↓
                    SonicWall TZ370 (WAN: <WAN_IP>)
                              ↓
                    Lab Network (10.0.10.0/24)
                    ├── GLaDOS    10.0.10.10  (SOC/SIEM)
                    ├── Aperture  10.0.10.20  (Proxmox/CTF)
                    ├── Pi 5      10.0.10.30  (Home Assistant)
                    └── Pi 4      10.0.10.40  (Pi-hole/Uptime Kuma)
```

## Hardware

| Machine | Specs | Role |
|---|---|---|
| ThinkCentre M70Q | i7-10700T, 32GB DDR4, 1TB SSD | SOC / SIEM (GLaDOS) |
| ThinkCentre M70Q | i7-10700T, 16GB DDR4, 500GB SSD | Proxmox / CTF (Aperture) |
| Raspberry Pi 5 | 8GB | Home Assistant OS |
| Raspberry Pi 4 | 4GB | Pi-hole, Uptime Kuma, Node-RED |
| SonicWall TZ370 | Gen 7 | Firewall / Gateway |

## Software Stack

### GLaDOS (SOC Machine)
- Ubuntu Server 24.04 LTS
- Wazuh 4.14.x (SIEM + XDR)
- Suricata (Network IDS)
- Zeek 8.0 (Network Traffic Analysis)
- Graylog (Log Aggregation) — planned
- TheHive + Cortex (Incident Response) — planned

### Aperture (Proxmox Machine)
- Proxmox VE (Trixie)
- Kali Linux VM (ID 100)
- Metasploitable 2 VM (ID 101)
- Suricata on vmbr0 (VM-to-VM monitoring)
- Wazuh Agent

## Setup Progress

- [x] SonicWall TZ370 configured
- [x] SSL-VPN working (NetExtender)
- [x] GLaDOS — Ubuntu + Wazuh + Suricata + Zeek
- [x] Aperture — Proxmox + Kali VM + Metasploitable VM
- [x] Wazuh agents on Aperture and Kali
- [x] First attack/detect exercise complete
- [ ] Graylog + TheHive on GLaDOS
- [ ] Pi 5 + Pi 4 setup
- [ ] Wazuh agents on Pis

## Security Note

All passwords and WAN-facing IPs have been redacted in this documentation.
Copy `.env.example` to `.env` and fill in your values.

**Never commit `.env` to a public repository.**

## Sessions

- [Session 01 - GLaDOS Setup](docs/Session%2001%20-%20GLaDOS%20Setup.md)
- [Session 02 - Aperture Setup](docs/Session%2002%20-%20Aperture%20Setup.md)

## Attack Labs

- [Attack 01 - vsftpd Metasploitable](docs/Attack%2001%20-%20vsftpd%20Metasploitable.md)
