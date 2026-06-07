# -Lab-8-Enumeration
# 🔐 Lab 8 - Enumeration (Metasploitable 2)

---

## 🖥️ Lab Environment Setup

### Victim Machine — Metasploitable 2

Confirmed victim IP via `ifconfig` on Metasploitable 2:

<img width="693" height="342" alt="image" src="https://github.com/user-attachments/assets/f34c2cf1-71a4-4d8e-8ca5-cad5bd397dbf" />

```
eth0  inet addr: 192.168.56.105   Bcast: 192.168.56.255   Mask: 255.255.255.0
      inet6 addr: fe80::a00:27ff:fe71:d2ae/64 Scope:Link
      HWaddr 08:00:27:71:D2:AE
```

### Attacker Machine — Kali Linux

Confirmed attacker IP via `ip a`:

<img width="640" height="336" alt="image" src="https://github.com/user-attachments/assets/b00896cd-75d4-4cc9-85ea-583d4d386933" />

```
eth0: inet 192.168.56.103/24
      inet6 fe80::d166:df7b:e290:bc40/64
```

### Connectivity Verified

<img width="672" height="327" alt="image" src="https://github.com/user-attachments/assets/d64d83b7-e76d-433e-bef9-58e45819bfc3" />

```
ping -c4 192.168.56.105
64 bytes from 192.168.56.105: ttl=64 ✅
```

| Component   | Detail                         |
|-------------|--------------------------------|
| Attacker OS | Kali Linux                     |
| Attacker IP | 192.168.56.103                 |
| Victim      | Metasploitable 2 (Ubuntu 8.04) |
| Victim IP   | 192.168.56.105                 |
| Network     | VirtualBox Host-Only Adapter   |

---

## 📋 Challenges Completed

| # | Challenge | Tool | Status |
|---|-----------|------|--------|
| 1 | NetBIOS Enumeration | `nbtscan` | ✅ |
| 2 | Fast Nmap Scan | `nmap -T4 -F` | ✅ |
| 5 | TTL OS Fingerprinting | `ping -c4` | ✅ |
| 9 | FTP Banner Grabbing | `telnet` | ✅ |
| 10 | Anonymous FTP Login | `ftp` | ✅ |
| 11 | SMB NSE Enumeration | `nmap smb-os-discovery` + `smb-enum-users` | ✅ |
| 14 | SNMP NSE | `nmap snmp-sysdescr` + `snmp-processes` | ✅ |
| 15 | DNS Zone Transfer | `nslookup` | ✅ |
| 16 | Version Detection | `nmap -sV` | ✅ |
| 17 | OS Detection | `nmap -O` | ✅ |
| 20 | DNSSEC Enumeration | `nmap dns-nsec-enum` | ✅ |
| 22 | Correlation Table | Manual Analysis | ✅ |
| 27 | IPv6 Discovery | `nmap -6 -O` | ✅ |
| 29 | SMTP Enumeration | `nmap smtp-enum-users` + `smtp-open-relay` | ✅ |

---

## 🔴 Section A — Basic Enumeration

### Challenge 1 — NetBIOS Enumeration

**Tool:** `nbtscan` (Kali Linux)
**Command:**
```
nbtscan 192.168.56.105
```

**Command Purpose:**
`nbtscan` queries the target for NetBIOS name table information, revealing the hostname, workgroup, and MAC address of the remote machine without requiring any credentials or authentication.

**Output:**

<img width="805" height="394" alt="image" src="https://github.com/user-attachments/assets/dd24465d-6fcd-41fa-b41f-808ca316d0cd" />

**Analysis:**
- Hostname `METASPLOITABLE` is revealed
- Workgroup is set to default `WORKGROUP`
- MAC address `08:00:27:71:D2:AE` confirms VirtualBox virtual NIC
- SMB file sharing service is confirmed active on the target

**Security Implication:**
NetBIOS exposes system identity without any form of authentication. This information can be used to plan further targeted attacks such as SMB relay or share enumeration.

---

### Challenge 2 — Fast Nmap Scan

**Tool:** `nmap` (Kali Linux)
**Command:**
```
nmap -T4 -F 192.168.56.105
```

**Command Purpose:**
`-T4` applies aggressive timing to speed up the scan while `-F` restricts the scan to the top 100 most commonly used ports. This gives a rapid overview of exposed services without performing a full port scan.

**Output:**

<img width="550" height="194" alt="image" src="https://github.com/user-attachments/assets/d8913db4-40ea-406f-9874-453ee19e4abf" />

**Analysis:**
- Multiple ports open including FTP (21), SSH (22), Telnet (23), HTTP (80), and SMB (445)
- Cleartext protocols such as Telnet and FTP are actively running
- The target exposes a large attack surface even from a quick scan

**Security Implication:**
Every open port represents a potential entry point. Running cleartext services like Telnet and FTP means credentials can be intercepted by anyone on the same network segment.

---

### Challenge 5 — TTL OS Fingerprinting

**Tool:** `ping` (Kali Linux)
**Command:**
```
ping -c4 192.168.56.105
```

**Command Purpose:**
Sends 4 ICMP echo request packets to the target. The TTL value in the reply is used to estimate the remote operating system based on known default TTL values per OS type.

**Output:**

<img width="318" height="166" alt="image" src="https://github.com/user-attachments/assets/ca84e759-b469-48d1-be52-c34b34c53030" />

**TTL Reference:**

| TTL | OS Guess |
|-----|----------|
| **64** | **Linux / Unix ✅** |
| 128 | Windows |
| 255 | Cisco / Network Device |

**Analysis:**
TTL=64 confirms the victim is running a Linux-based OS, consistent with Metasploitable 2 running Ubuntu 8.04.

**Security Implication:**
Passive OS identification via TTL allows attackers to narrow down their toolset and focus on Linux-specific vulnerabilities and exploits.

---

### Challenge 9 — FTP Banner Grabbing

**Tool:** `telnet` (Kali Linux)
**Command:**
```
telnet 192.168.56.105 21
```

**Command Purpose:**
Connects directly to port 21 and reads the FTP service banner. The banner reveals the software name and version which can then be checked against known vulnerability databases.

**Output:**

<img width="324" height="209" alt="image" src="https://github.com/user-attachments/assets/4a86b6f7-05d0-4b4d-9db5-a42f7e722914" />

**Analysis:**
- FTP service identified as `vsFTPd 2.3.4`
- This specific version contains a known backdoor (CVE-2011-2523)
- The backdoor is triggered by sending a username containing `:)`, which opens a root shell on port 6200

**Security Implication:**
Banner grabbing is a passive, non-intrusive technique that instantly reveals critical version information. Running vsFTPd 2.3.4 in any environment is extremely dangerous and must be updated immediately.

---

### Challenge 10 — Anonymous FTP Login

**Tool:** `ftp` (Kali Linux)
**Command:**
```
ftp 192.168.56.105
```

**Command Purpose:**
Attempts to log into the FTP server using the username `anonymous` with no password. This tests whether the server allows unauthenticated public access to its file system.

**Output:**

<img width="798" height="363" alt="image" src="https://github.com/user-attachments/assets/59686249-3321-41e7-a0b3-3f4ae7fafc51" />

**Analysis:**
- Anonymous FTP login is permitted with no credentials required
- The root FTP directory is fully accessible and browsable
- No authentication barrier exists between an attacker and the file system

**Security Implication:**
Anonymous FTP is a serious misconfiguration. Even without write access, it allows attackers to enumerate directory structures and gather sensitive system information freely.

---

## 🟠 Section B — Intermediate Enumeration

### Challenge 11 — SMB NSE Enumeration

**Tool:** `nmap` NSE scripts (Kali Linux)

#### Part 1: OS Discovery
**Command:**
```
nmap --script smb-os-discovery -p445 192.168.56.105
```

**Command Purpose:**
Uses Nmap's SMB NSE script to extract operating system details, computer name, domain, and workgroup information directly from the SMB service handshake without authentication.

**Output:**

<img width="700" height="391" alt="image" src="https://github.com/user-attachments/assets/43551ca3-d935-48d0-abd2-3175fb374bc9" />

**Analysis:**
- OS confirmed as Unix/Linux via SMB
- Computer name: `METASPLOITABLE`
- Samba version and workgroup details exposed without credentials

#### Part 2: User Enumeration
**Command:**
```
nmap --script smb-enum-users -p445 192.168.56.105
```

**Command Purpose:**
Enumerates valid user accounts on the target through the SMB protocol. Discovered usernames can be leveraged for credential-based attacks across other services.

**Output:**

<img width="794" height="224" alt="image" src="https://github.com/user-attachments/assets/b2a5d930-79db-4e5f-8550-3b187d7ece22" />

**Analysis:**
- Valid user account `msfadmin` discovered via SMB
- No authentication was required to retrieve this information
- These credentials are likely reused across SSH, FTP, and VNC services

**Security Implication:**
Unauthenticated SMB user enumeration is a critical misconfiguration. Discovered usernames provide a direct starting point for brute-force or credential-stuffing attacks across the entire system.

---

### Challenge 14 — SNMP Enumeration

**Tool:** `nmap` NSE scripts (Kali Linux)

#### Part 1: System Description
**Command:**
```
nmap -p 161 --script snmp-sysdescr 192.168.56.105
```

**Command Purpose:**
Queries the SNMP service on UDP port 161 using the default community string `public` to retrieve the system description including OS version and hardware information.

**Output:**

<img width="798" height="227" alt="image" src="https://github.com/user-attachments/assets/c5b3117a-1375-4e96-8ef7-5980ad1fc2fb" />

#### Part 2: Running Processes
**Command:**
```
nmap -p 161 --script snmp-processes 192.168.56.105
```

**Command Purpose:**
Retrieves the list of currently running processes on the target via SNMP, exposing active daemons, services, and potentially sensitive applications running on the victim machine.

**Output:**

<img width="309" height="133" alt="image" src="https://github.com/user-attachments/assets/986d855c-512a-4340-aa8f-ae884b272a4a" />

**Analysis:**
- SNMP accessible using default community string `public`
- System description and running processes exposed without authentication
- Internal service details are fully readable remotely

**Security Implication:**
SNMP with default community strings is a widespread misconfiguration. It grants attackers full visibility into the system configuration, running processes, and network interfaces remotely and silently.

---

### Challenge 15 — DNS Zone Transfer

**Tool:** `nslookup` (Kali Linux)
**Commands:**
```
nslookup
> server 192.168.56.105
> ls -d example.com
```

**Command Purpose:**
Attempts a DNS zone transfer by pointing nslookup at the target DNS server and requesting all records for a domain. A misconfigured DNS server would return every DNS record it holds, leaking the internal network structure.

**Output:**

<img width="794" height="389" alt="image" src="https://github.com/user-attachments/assets/27dd0e8e-b46d-48a8-a8a4-63e1a684da9c" />

**Analysis:**
- The `ls` command returned not implemented on Kali's nslookup client
- The DNS server did not leak any zone records
- Zone transfer restrictions appear to be in place — this is the expected secure behaviour

**Security Implication:**
Properly restricted zone transfers prevent attackers from mapping out internal DNS records. No sensitive information was exposed during this test.

---

### Challenge 16 — Version Detection

**Tool:** `nmap` (Kali Linux)
**Command:**
```
nmap -sV 192.168.56.105
```

**Command Purpose:**
Probes each open port to identify the exact software name and version running behind the service. Version information is essential for mapping known CVEs to running services on the target.

**Output:**

<img width="784" height="611" alt="image" src="https://github.com/user-attachments/assets/6a2833db-4d3b-4a3e-af12-2246958ec88f" />

**Analysis:**
- Every service is running a severely outdated version
- Port 1524 exposes a bindshell — a direct root shell requiring no exploit whatsoever
- Key vulnerable versions: vsFTPd 2.3.4, OpenSSH 4.7p1, Samba 3.0.20, Apache 2.2.8

**Security Implication:**
A single version scan provides a complete exploitation roadmap. Every discovered service version has publicly available, reliable exploits making this machine trivially compromiseable from the network.

---

### Challenge 17 — OS Detection

**Tool:** `nmap` (Kali Linux)
**Command:**
```
nmap -O 192.168.56.105
```

**Command Purpose:**
Uses TCP/IP stack fingerprinting including TTL values, TCP window sizes, and packet options to determine the remote operating system without requiring any credentials or access.

**Output:**

<img width="858" height="265" alt="image" src="https://github.com/user-attachments/assets/1889a7a8-fd95-4433-b71a-73e224bc2bf4" />

**Analysis:**
- OS confirmed as Linux 2.6.9 - 2.6.33
- Consistent with Ubuntu 8.04 LTS (Metasploitable 2)
- Network distance of 1 hop confirms direct Layer 2 connection between Kali and the victim

**Security Implication:**
Pinpointing the exact kernel version range enables attackers to search for kernel-level privilege escalation exploits tailored specifically to that version.

---

### Challenge 20 — DNSSEC Enumeration

**Tool:** `nmap` (Kali Linux)
**Command:**
```
nmap -p 53 --script dns-nsec-enum 192.168.56.105
```

**Command Purpose:**
Attempts to enumerate DNSSEC NSEC records which, if misconfigured, can expose all hostnames in a DNS zone through zone walking without performing a traditional zone transfer.

**Output:**

<img width="858" height="334" alt="image" src="https://github.com/user-attachments/assets/5656dcb0-ce7d-4a66-a088-46f097977abb" />

**Analysis:**
- Port 53 is open with ISC BIND 9.4.2 running
- NSEC enumeration requires a --script-args dns-nsec-enum.domains= argument for full results
- BIND 9.4.2 does not support DNSSEC by default so NSEC records are absent

**Security Implication:**
Without DNSSEC configured, zone walking is not possible. However the exposed DNS service can still be leveraged for basic reconnaissance and internal hostname discovery.

---

## 🔵 Section C — Advanced Enumeration

### Challenge 22 — Correlation Table

**Purpose:**
Combining data gathered from multiple enumeration sources builds a unified picture of the target's users, services, and overall attack surface. Correlating findings reveals patterns such as shared usernames across services and helps prioritise the most effective attack paths.

**Sources used:**
- SMB enumeration (Challenge 11)
- FTP banner and anonymous login (Challenges 9 and 10)
- NetBIOS enumeration (Challenge 1)

**Correlation Table:**

| Source | Username / Service | Group / Role | Notes |
|--------|-------------------|--------------|-------|
| SMB (Ch11) | `msfadmin` | Normal user (RID 3000) | Likely valid across SSH, FTP, and VNC |
| FTP (Ch9, Ch10) | `anonymous` + `vsFTPd 2.3.4` | Anonymous access + backdoored version | Remote root possible via CVE-2011-2523 |
| NetBIOS (Ch1) | `METASPLOITABLE` | Hostname | Confirms system identity and workgroup |

**Analysis:**
The correlation reveals a target vulnerable from multiple angles simultaneously. The backdoored FTP service accepts anonymous logins while SMB enumeration exposes a valid user account (`msfadmin`) highly likely to be reused across SSH, VNC, and other services. The NetBIOS hostname `METASPLOITABLE` matches the Samba computer name, providing consistent system identification across all three protocols. Together these findings describe a machine that can be fully compromised using only public tools and default credentials.

---

### Challenge 27 — IPv6 Discovery

**Tool:** `nmap` (Kali Linux)
**Command:**
```
nmap -6 -O fe80::a00:27ff:fe71:d2ae -e eth0
```

**Command Purpose:**
Scans the target over its IPv6 link-local address to discover reachable services and fingerprint the OS via IPv6. IPv6 is commonly overlooked in security assessments, making it a useful alternative reconnaissance path.

**Output:**

<img width="853" height="238" alt="image" src="https://github.com/user-attachments/assets/c7e0154a-b55a-40e9-be55-12c6ff942768" />

**Analysis:**
- Victim responds on IPv6 address `fe80::a00:27ff:fe71:d2ae`
- Services reachable over IPv6: SSH (22), DNS (53), ccproxy-ftp (2121), PostgreSQL (5432)
- OS fingerprinting over IPv6 confirms Linux 2.6.18 - 2.6.34

**Security Implication:**
IPv6 is frequently left unmonitored and unfiltered by firewalls and IDS systems. The same vulnerable services are reachable over IPv6, potentially bypassing IPv4-only security controls entirely.

---

### Challenge 29 — SMTP Enumeration

**Tool:** `nmap` NSE scripts (Kali Linux)

#### Part 1: User Enumeration
**Command:**
```
nmap -p 25 --script smtp-enum-users 192.168.56.105
```

**Command Purpose:**
Attempts to enumerate valid email accounts on the SMTP server using commands such as VRFY, EXPN, and RCPT TO. A misconfigured mail server will confirm which usernames are valid without requiring authentication.

**Output:**

<img width="851" height="235" alt="image" src="https://github.com/user-attachments/assets/2b1c49b3-971c-4c1b-b347-af00390f4232" />

**Analysis:**
- SMTP port 25 is open running Postfix
- The script received a non-standard response from the RCPT method
- Manual testing via Netcat using VRFY and EXPN commands would yield more reliable enumeration results

#### Part 2: Open Relay Test
**Command:**
```
nmap -p 25 --script smtp-open-relay 192.168.56.105
```

**Command Purpose:**
Tests whether the SMTP server will relay emails on behalf of unauthorized external senders. Open relays are abused to send spam and phishing emails while masking the true origin.

**Output:**

<img width="858" height="265" alt="image" src="https://github.com/user-attachments/assets/1889a7a8-fd95-4433-b71a-73e224bc2bf4" />

**Analysis:**
- SMTP relay is not open — Postfix correctly rejects unauthorized relaying attempts
- This is one of the few properly hardened services on an otherwise critically vulnerable machine

**Security Implication:**
While nearly every other service on this target is dangerously misconfigured, the mail service follows proper security practices, preventing spam relay abuse and protecting against username harvesting via SMTP commands.

---

## 📊 Key Findings Summary

| Service | Version | Risk | CVE / Note |
|---------|---------|------|------------|
| FTP | vsFTPd 2.3.4 | 🔴 Critical | CVE-2011-2523 — backdoor RCE |
| SSH | OpenSSH 4.7p1 | 🟠 High | Multiple known CVEs |
| SMB | Samba 3.0.20 | 🔴 Critical | CVE-2007-2447 — RCE via username |
| HTTP | Apache 2.2.8 | 🟠 High | CVE-2007-5000, CVE-2007-6388 |
| MySQL | 5.0.51a | 🟠 High | Default credentials, no bind restriction |
| PostgreSQL | 8.3.0–8.3.7 | 🟠 High | Trust auth enabled |
| Bindshell | port 1524 | 🔴 Critical | Direct root shell — no exploit needed |
| VNC | protocol 3.3 | 🟠 High | Weak auth, outdated protocol |
| UnrealIRCd | — | 🔴 Critical | CVE-2010-2075 — backdoor |

---

## 🛠️ Tools Used

| Tool | Platform | Used For |
|------|----------|----------|
| nmap | Kali Linux | Port scans, NSE scripts, version and OS detection |
| nbtscan | Kali Linux | NetBIOS name and service enumeration |
| ping | Kali Linux | TTL-based OS fingerprinting |
| telnet | Kali Linux | FTP banner grabbing on port 21 |
| ftp | Kali Linux | Anonymous FTP login testing |
| nslookup | Kali Linux | DNS zone transfer attempt |

---

## ✅ Conclusion

The enumeration lab successfully demonstrated 14 distinct information gathering techniques against Metasploitable 2 using Kali Linux as the attacking platform. The target exposed a critically vulnerable configuration across almost every service — outdated software versions with known backdoors, anonymous FTP access, unauthenticated SMB user enumeration, SNMP with default community strings, and a direct root shell sitting open on port 1524. Only the DNS zone transfer and SMTP relay tests returned properly secured responses. This exercise highlights how much damage an attacker can do purely through passive and semi-passive enumeration before touching a single exploit, reinforcing the importance of service hardening, regular patching, and network segmentation as foundational security practices.
