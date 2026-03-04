# Screenshots Index

Place your screenshots in this folder following the naming convention below.

## Naming Convention

```
[phase]-[step]-[description].png
```

## Required Screenshots

### Phase 1: Reconnaissance
| Filename | Description |
|----------|-------------|
| `01-recon-rustscan.png` | RustScan open ports |
| `02-recon-enum4linux.png` | enum4linux-ng domain info |
| `03-recon-ldapsearch-dump.png` | Anonymous LDAP bind |
| `04-recon-ldapsearch-users.png` | Filtered users with description |
| `05-recon-rid-brute.png` | CrackMapExec RID brute results |
| `06-recon-realusers.png` | realusers.txt content |
| `07-recon-guest.png` | Guest account confirmed |

### Phase 2: Initial Access
| Filename | Description |
|----------|-------------|
| `08-access-asrep-hashes.png` | AS-REP Roasting hashes (dead end) |
| `09-access-kerberoast.png` | Kerberoasting via Guest, CODY_ROY |
| `10-access-john-codyroy.png` | John cracking CODY_ROY hash |
| `11-access-rdp-codyroy.png` | RDP desktop as CODY_ROY |

### Phase 3: Lateral Movement
| Filename | Description |
|----------|-------------|
| `12-lateral-password-spray.png` | ZACHARY_HUNT password reuse |
| `13-lateral-bloodhound-collect.png` | bloodhound-python collection |
| `14-lateral-bloodhound-path.png` | BloodHound GenericWrite path |
| `15-lateral-genericwrite-info.png` | BloodHound abuse info |
| `16-lateral-bloodyad-spn.png` | bloodyAD fake SPN set |
| `17-lateral-targeted-kerberoast.png` | JERRI_LANCASTER Kerberoastable |
| `18-lateral-john-jerri.png` | John cracking JERRI hash |
| `19-lateral-rdp-jerri.png` | RDP desktop as JERRI_LANCASTER |

### Phase 4: Domain Compromise
| Filename | Description |
|----------|-------------|
| `20-priv-syncer-script.png` | syncer.ps1 hardcoded credentials |
| `21-priv-bloodhound-sanford.png` | SANFORD_DAUGHERTY → Domain Admins path |
| `22-priv-crackmapexec-pwn3d.png` | CrackMapExec Pwn3d! confirmation |
| `23-priv-secretsdump.png` | NTDS dump, Administrator hash |
| `24-priv-smbclient.png` | smbclient C$ access |
| `25-flag.png` | Flag retrieved |