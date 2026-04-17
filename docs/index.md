# Offensive Security & Engineering Portfolio

Hi. I'm a Cyber Engineer, or at least that's what my degree says. I occasionally develop stuff and perform pentesting exercises, practicing on the reporting side of things, since I've gathered that's what companies actually care about.

Here you will find a collection of **simulated assessments** based on realistic scenarios: AD compromise via MS-SQL, certificate-based escalation, web application injection, cryptographic exploitation. These are not real engagements (for legal reasons 🙃), but they get the point across well. 

## Why this approach to reporting?

In penetration testing, finding the vulnerability is only 50% of the job. The other 50%? Communicating the impact to stakeholders, who may not always understand the technical jargon. **Therefore, report != technical POC, but narrative + findings + remediation** that infrastructure teams can actually implement. I even wrote a thesis on automating this, but I'm not publishing it so it doesn't get stolen; obviously, if you're interested I can send it over, but you never know.

---

## The Reports

### 🔴 **Offensive Security Assessments**

- **["Imagery" Scenario](imagery.md)** – Web Application Pentest (OWASP)  
  Blind XSS → LFI → Command Injection. How an image gallery becomes an entry point for RCE. CVSS 9.8.

- **["Expressway" Scenario](expressway.md)** – Network & Cryptography (NIST 800-115)  
  VPN attack: IKE Phase 1 enumeration → Offline PSK cracking → privilege escalation via Sudo CVE. CVSS 9.2.

- **["DarkZero" Scenario](darkzero.md)** – Active Directory & Infrastructure (OSSTMM)  
  AD domain compromise starting from a standard user: LDAP enumeration → AD CS misconfiguration → ESC1 certificate forgery → Domain Admin. CVSS 9.9.

- **["Signed" Scenario](signed.md)** – Kerberos Abuse & MS-SQL (PTES)  
  Exposed DB → SMB auth coercion → NTLM hash extraction → Silver Ticket forging via PAC manipulation. Two ticket stages, two layers of escalation. CVSS 9.8.

### ⚙️ **Projects**

- **[LazyPwn – Asynchronous CTF Orchestrator](lazypwn.md)** – An asynchronous, event-driven framework written in Python 3, designed to automate the reconnaissance phase during CTFs. It cuts down scanning times, handles network interruptions via a stateful design, and provides post-exploitation support routines.

- **[Anime-themed gallery – Boot2Root Design](eth.md)** – Design of an Ubuntu Server VM hosting a controlled vulnerable web app for educational purposes. 6 vulnerabilities, 3 difficulty levels, progressive exploit chain.

---

## Topics, Technologies & Skills

| Area | Methodologies & Tools |
|------|-------------------------|
| **AD & Kerberos** | PTES, OSSTMM, Silver/Golden Ticket, PAC manipulation, GPO analysis |
| **Web Security** | OWASP Top 10, XSS (blind), LFI, command injection, CSRF, authentication bypass |
| **Network & Crypto** | NIST 800-115, IKE/IPSec, PSK cracking, certificate exploitation, WinRM |
| **Tools** | Impacket suite, Hashcat, Burp Suite, Bloodhound, Certipy, Responder, Nmap, Metasploit |
| **Reporting** | CVSS v3.1, PTES Standard, executive summary, remediation roadmap, threat modeling |

---

## Quick Notes

It's dangerous to go alone. Take this. 🍺

---

**Disclaimer:** The reports are sanitized – no real credentials, no compromised public IPs. All data comes from isolated labs (HackTheBox, HTB Season 9, custom VMs).
---
