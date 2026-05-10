# 🏢 Active Directory
## Part 1: Foundation + No-Credential Enumeration

> **Target Setup:** Kali Linux (Attacker) → DC01 + WS01 + WS02
> **Goal:** Bina credentials ke maximum information nikalna

---

## 📋 Quick Checklist — Part 1 Complete karne ke liye

```
[ ] Domain name pata chali
[ ] DC IP confirm hua
[ ] Open ports list bani + saved
[ ] /etc/hosts updated
[ ] SMB shares list mili
[ ] Password policy noted — lockout threshold critical hai
[ ] User list mili (rpcclient / enum4linux-ng se)
```

---

## 🗺️ AD Structure — Quick Reference

```
Forest: corp.com
└── Domain: corp.local
    ├── DC01 (Domain Controller) ← 👑 Sabse important machine
    ├── WS01 (Workstation)
    └── WS02 (Workstation)

Attack Chain:
Foothold WS01 → Privesc → Creds dump → Lateral WS02 → DC Compromise
```

---

## STEP 1 — Host Discovery

### Ping Sweep
```bash
nmap -sn 192.168.1.0/24
```

### ARP Scan (jab ICMP block ho)
```bash
sudo arp-scan -l
sudo arp-scan 192.168.1.0/24
```

### NetBIOS Scan — Windows machines + names
```bash
nbtscan 192.168.1.0/24
```

**Output padhna:**
```
192.168.1.10    CORP\DC01    SHARING DC    ← "DC" tag = Domain Controller
192.168.1.20    CORP\WS01    SHARING
192.168.1.21    CORP\WS02    SHARING
```

---

## STEP 2 — Port Scanning

### Quick Scan (top ports)
```bash
nmap -sC -sV --open -T4 192.168.1.10
```

### Full Scan — OSCP exam ke liye (saare 65535 ports)
```bash
nmap -sC -sV -p- --open -T4 -oN dc_full_scan.txt 192.168.1.10
```

### Flags explained
| Flag | Matlab |
|------|--------|
| `-sC` | Default NSE scripts chalao (smb-os-discovery, etc.) |
| `-sV` | Service version detect karo |
| `-p-` | Saare 65535 ports scan karo |
| `--open` | Sirf open ports dikhao |
| `-T4` | Speed aggressive (exam ke liye) |
| `-oN` | Normal format file mein save karo |

### Speed vs Accuracy
| Situation | Command |
|-----------|---------|
| Fast check | `nmap -sC -sV --open -T4 IP` |
| Full scan (exam) | `nmap -sC -sV -p- --open -T4 -oN out.txt IP` |
| Stealth | `nmap -sS -T2 -p- IP` |

---

## STEP 3 — AD Ports Cheatsheet

| Port | Service | Khula hai matlab... | Attack Surface |
|------|---------|---------------------|----------------|
| **53** | DNS | **DC hai** (99% sure) | Zone transfer, DNS recon |
| **88** | Kerberos | **DC confirmed** | ASREPRoast, Kerberoast |
| **135** | RPC | Windows RPC chal rahi | Null session, user enum |
| **139/445** | SMB | File sharing on | Null session, relay attacks |
| **389/636** | LDAP/LDAPS | Directory exposed | Anonymous bind, user enum |
| **464** | Kpasswd | Password change service | Brute force (no lockout pe) |
| **3268/3269** | Global Catalog | Large/multi-domain AD | Cross-domain queries |
| **3389** | RDP | Remote Desktop on | Password spray, cred login |
| **5985** | WinRM | PowerShell remoting | Evil-WinRM shell |

> 🎯 **Rule:** Port 53 + 88 + 389 ek saath = **Confirmed Domain Controller**

---

## STEP 4 — Domain Name + /etc/hosts Setup

### Nmap output mein domain name kahan dikhta hai
```
| smb-os-discovery:
|   Computer name: DC01
|   Domain name: corp.local          ← YAHAN
|   FQDN: DC01.corp.local            ← YAHAN BHI
```

### /etc/hosts update karo
```bash
sudo nano /etc/hosts
```

```
# AD Lab
192.168.1.10    DC01.corp.local DC01 corp.local
192.168.1.20    WS01.corp.local WS01
192.168.1.21    WS02.corp.local WS02
```

> **Kyun zaroori:** Kerberos IP se nahi, naam se kaam karta hai. Bina yeh entry ke tools fail honge.

---

## STEP 5 — SMB Enumeration (Port 445)

### SMB Signing check — important!
```
signing:True  → NTLM Relay mushkil (DC pe hota hai)
signing:False → NTLM Relay possible! (Workstations pe hota hai)
```

### Anonymous / Null Session
```bash
# Null session — bina credentials ke
smbclient -N -L //192.168.1.10

# smbmap anonymous
smbmap -H 192.168.1.10

# smbmap guest (blank password)
smbmap -u guest -p "" -H 192.168.1.10
```

**Output example:**
```
Sharename       Type    Comment
---------       ----    -------
ADMIN$          Disk    Remote Admin
C$              Disk    Default share
IPC$            IPC     Remote IPC
SYSVOL          Disk    Logon server share   ← IMPORTANT
NETLOGON        Disk    Logon server share   ← IMPORTANT
SharedFiles     Disk    Company Files        ← CUSTOM = ZAROOR DEKHO
```

### Shares ke andar dekho
```bash
# Recursive listing
smbmap -H 192.168.1.10 -r SYSVOL

# smbclient se connect + browse
smbclient -N //192.168.1.10/SYSVOL
smb: \> recurse on
smb: \> ls

# File download
smb: \> get filename.txt
```

### NetExec / CrackMapExec se SMB scan
```bash
# Kaun sa installed hai check karo
which nxc || which crackmapexec

# Poore subnet ka SMB scan
nxc smb 192.168.1.0/24
crackmapexec smb 192.168.1.0/24

# Shares list
nxc smb 192.168.1.10 --shares

# Null session shares
nxc smb 192.168.1.10 -u '' -p '' --shares

# Guest session shares
nxc smb 192.168.1.10 -u 'guest' -p '' --shares
```

**Output padhna:**
```
SMB  192.168.1.10  445  DC01  [*] Windows Server 2019 (name:DC01) (domain:corp.local) (signing:True)
SMB  192.168.1.20  445  WS01  [*] Windows 10 x64    (name:WS01) (domain:corp.local) (signing:False)
```

### Interesting Files dhundna
```bash
# Mount karke grep karo
sudo mkdir /mnt/smb
sudo mount -t cifs //192.168.1.10/SharedFiles /mnt/smb -o guest

grep -ri "password" /mnt/smb/
grep -ri "passwd" /mnt/smb/
grep -ri "pwd" /mnt/smb/
grep -ri "secret" /mnt/smb/
```

**Interesting extensions:**
| Extension | Kyun |
|-----------|------|
| `.txt` | Notes, passwords |
| `.xml` | Config — GPP passwords |
| `.ini` | App configs |
| `.config` | Web app — DB creds |
| `.bat` / `.ps1` | Scripts — hardcoded creds |
| `.kdbx` | KeePass database |
| `id_rsa` | SSH private key |

### SYSVOL — GPP Password (Classic Find)
```bash
# SYSVOL browse karo Groups.xml dhundo
smbclient -N //192.168.1.10/SYSVOL
smb: \> recurse on
smb: \> ls

# Groups.xml mila to download karo
smb: \> get corp.local/Policies/{GUID}/User/Groups.xml

# Decrypt karo
gpp-decrypt "ENCRYPTED_STRING_HERE"
```

### Common SMB Errors
| Error | Matlab | Fix |
|-------|--------|-----|
| `NT_STATUS_ACCESS_DENIED` | Anonymous allowed nahi | Credentials try karo |
| `NT_STATUS_LOGON_FAILURE` | Wrong credentials | user/pass check karo |
| `NT_STATUS_CONNECTION_REFUSED` | Port band | `nmap -p 445 IP` confirm karo |
| Connection timeout | Machine unreachable | Ping + hosts file check |

---

## STEP 6 — RPC Enumeration (Port 135)

### Null Session
```bash
rpcclient -U "" -N 192.168.1.10
```

### Andar jaake yeh commands chalao
```bash
# Saare users
rpcclient $> enumdomusers

# Saare groups
rpcclient $> enumdomgroups

# Domain info + password policy
rpcclient $> querydominfo

# Password policy details
rpcclient $> getdompwinfo

# User details
rpcclient $> querydispinfo

# Specific user info (RID se)
rpcclient $> queryuser 0x44f
```

**enumdomusers output:**
```
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[john.doe] rid:[0x44f]
user:[svc_backup] rid:[0x451]    ← Service accounts = priority targets
```

> **Note karo:** Service accounts (`svc_*`) pe weak passwords hone ki probability zyada hoti hai.

### Password Policy — CRITICAL ⚠️

```bash
rpcclient $> getdompwinfo
```

```
min_password_length: 7
password_properties: DOMAIN_PASSWORD_COMPLEX
```

**Lockout Strategy:**
| Lockout Threshold | Strategy |
|-------------------|----------|
| **0** | Lockout nahi — full spray karo |
| **3** | Max **2 passwords** per account |
| **5** | Max **3-4 passwords**, phir ruko |
| **10+** | 7-8 passwords safe hain |

> 🔴 **GOLDEN RULE:** Spray se PEHLE lockout check KARO. Account lock = exam barbad.

---

## STEP 7 — enum4linux-ng (All-in-One)

```bash
# Full enumeration
enum4linux-ng -A 192.168.1.10

# Save output
enum4linux-ng -A 192.168.1.10 | tee enum4linux_output.txt

# Sirf users
enum4linux-ng -U 192.168.1.10

# Sirf shares
enum4linux-ng -S 192.168.1.10

# Sirf password policy
enum4linux-ng -P 192.168.1.10
```

**Output mein kya dhundna hai:**
```
[+] Domain SID: S-1-5-21-XXXXXXXXXX     ← Note karo
[+] john.doe (RID 1103)                  ← Users
[+] svc_backup (RID 1105)               ← Service accounts!
[+] Minimum password length: 7
[+] Account lockout threshold: 3         ← SABSE IMPORTANT
[+] SYSVOL - READ ONLY                   ← Accessible share
```

---

## STEP 8 — LDAP Enumeration (Port 389)

```bash
# Anonymous LDAP query
ldapsearch -x -H ldap://192.168.1.10 -b "DC=corp,DC=local"

# Users nikalna
ldapsearch -x -H ldap://192.168.1.10 -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName

# Computers nikalna
ldapsearch -x -H ldap://192.168.1.10 -b "DC=corp,DC=local" "(objectClass=computer)"
```

> `-x` = simple auth (anonymous), `-H` = host, `-b` = base DN (domain ko DC=corp,DC=local format mein)

---

## STEP 9 — DNS Recon (Port 53)

```bash
# Domain ke saare records
dig ANY corp.local @192.168.1.10

# Zone transfer try karo (agar allow ho — goldmine!)
dig axfr corp.local @192.168.1.10

# DC ka IP confirm karo
nslookup corp.local 192.168.1.10

# Reverse lookup
dig -x 192.168.1.10 @192.168.1.10
```

**Zone transfer agar kaam kare:**
```
corp.local.    SOA    DC01.corp.local.
DC01           A      192.168.1.10
WS01           A      192.168.1.20
WS02           A      192.168.1.21
```
Poori network map mil gayi!

---

## 📝 Notes Template — OSCP Report Ready

```markdown
# AD Enumeration — [Target/Lab Name]
Date: [DATE]
Tester: [YOUR NAME]

---

## Network Map
| Hostname | IP | Role | OS |
|----------|----|------|----|
| DC01 | 192.168.1.10 | Domain Controller | Windows Server 2019 |
| WS01 | 192.168.1.20 | Workstation | Windows 10 |
| WS02 | 192.168.1.21 | Workstation | Windows 10 |

Domain: corp.local
Forest: corp.local

---

## Open Ports — DC01 (192.168.1.10)
| Port | Service | Notes |
|------|---------|-------|
| 53 | DNS | DC confirmed |
| 88 | Kerberos | DC confirmed |
| 135 | RPC | Null session: YES/NO |
| 445 | SMB | Signing: Enabled, Null: YES/NO |
| 389 | LDAP | Anonymous bind: YES/NO |
| 3389 | RDP | Open |
| 5985 | WinRM | Open |

---

## SMB Shares
| Share | Access | Files Found |
|-------|--------|-------------|
| SYSVOL | READ | Groups.xml — [YES/NO] |
| NETLOGON | READ | Scripts — [filenames] |
| SharedFiles | READ/WRITE | passwords.txt — creds found |

---

## Password Policy ⚠️
- Lockout Threshold: ___  ← CRITICAL
- Min Password Length: ___
- Complexity: YES/NO
- SPRAY STRATEGY: Max ___ passwords per account

---

## Users Found
```
john.doe
jane.smith
svc_backup        ← SERVICE ACCOUNT — priority
administrator
```

---

## Credentials Found
| Username | Password | Source | Access Level |
|----------|----------|--------|--------------|
| svc_backup | Password123! | SYSVOL/Groups.xml | Domain User |

---

## Next Steps
- [ ] ASREPRoasting — users without pre-auth
- [ ] Kerberoasting — service accounts
- [ ] Password spray (max ___ attempts — lockout=___)
- [ ] SYSVOL GPP check
- [ ] BloodHound collection
```

---

## 🔧 Tools Quick Reference

| Tool | Use | Install |
|------|-----|---------|
| `nmap` | Port scan | Pre-installed Kali |
| `smbclient` | SMB browse + download | Pre-installed Kali |
| `smbmap` | SMB permissions check | Pre-installed Kali |
| `nxc` / `crackmapexec` | SMB/RDP/WinRM scan | `pip install netexec` |
| `rpcclient` | RPC null session | Pre-installed Kali |
| `enum4linux-ng` | All-in-one enum | `apt install enum4linux-ng` |
| `nbtscan` | NetBIOS names | `apt install nbtscan` |
| `ldapsearch` | LDAP queries | Pre-installed Kali |
| `gpp-decrypt` | GPP password decrypt | Pre-installed Kali |

---

## ⚡ One-Liner Attack Sequence — No Credentials

```bash
# 1. Host discovery
nmap -sn 192.168.1.0/24

# 2. DC identify + full scan
nmap -sC -sV -p- --open -T4 -oN dc_scan.txt 192.168.1.10

# 3. /etc/hosts update (domain name nmap output se lo)
echo "192.168.1.10 DC01.corp.local DC01 corp.local" | sudo tee -a /etc/hosts

# 4. NetBIOS names
nbtscan 192.168.1.0/24

# 5. SMB — domain + signing info
nxc smb 192.168.1.0/24

# 6. SMB shares anonymous
nxc smb 192.168.1.10 -u '' -p '' --shares
smbclient -N -L //192.168.1.10
smbmap -H 192.168.1.10

# 7. SYSVOL browse
smbclient -N //192.168.1.10/SYSVOL

# 8. RPC — users + password policy
rpcclient -U "" -N 192.168.1.10 -c "enumdomusers"
rpcclient -U "" -N 192.168.1.10 -c "getdompwinfo"

# 9. enum4linux-ng — all in one
enum4linux-ng -A 192.168.1.10 | tee enum4linux_dc.txt

# 10. LDAP anonymous
ldapsearch -x -H ldap://192.168.1.10 -b "DC=corp,DC=local" | tee ldap_anon.txt

# 11. DNS zone transfer try
dig axfr corp.local @192.168.1.10
```

---

## 🚨 Common Mistakes — OSCP mein mat karna

1. **Password spray bina policy check kiye** → accounts lock ho jaate hain
2. **DC ka naam hosts mein nahi daala** → Kerberos attacks fail honge
3. **SYSVOL nahi dekha** → GPP passwords miss ho jaate hain
4. **Service accounts ignore kiye** → ye easy targets hote hain
5. **enum4linux output nahi save kiya** → dobara karna padega

---

*Part 1 Complete — Next: Part 1B (LDAP Deep Dive + BloodHound)*
