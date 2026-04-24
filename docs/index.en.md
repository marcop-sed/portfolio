# Offensive Security & Engineering Portfolio

I am a Cyber Engineer with horizontal knowledge and the goal of verticalizing more on DevSecOps and Offensive Security. I created this portfolio to keep track of my work.

Currently, the menu offers:

- **Simulated assessments**: Professional-style reports on HackTheBox scenarios, used to learn methodologies and get familiar with the standards used. AD compromise via MS-SQL, certificate-based escalation, web application injection, are some examples of what has been addressed. They are not real engagements (for legal reasons 🙃), but they get the point across well and serve the purpose.

- **Personal projects**: Development projects both in the cybersecurity field and not. Most of these arise from needs, to simplify and automate processes.



### Why write Reports on HTB machines

I was participating in Season 9 of HackTheBox to learn and experiment with Penetration Testing techniques; later, I was told they could be an excellent simulation to deepen reporting techniques, learning the methodologies related to the NIST 800-115, OSSTMM, and PTES frameworks. 

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
  Exposed DB → SMB auth coercion → NTLM hash extraction → Silver Ticket forging via PAC manipulation. Two ticket stages, two levels of escalation. CVSS 9.8. 


## ⚙️ **Projects**

### **Cybersecurity**

- **NERVA: Nexus for Enterprise Risk and Vulnerability Assessment** - *Master's Thesis Project*
  NERVA was born following an internship in the Purple Team field for research. The idea was to create a tool that takes a Red Team report as input and, through a correlation engine that hooks into an LLM, maps the vulnerabilities onto the NIST CSF 2.0 framework and the MITRE ATT&CK matrix; using the latter, it identifies the respective remediation proposed by D3FEND and proposes an applicable patch, obtained and refined by an internal learning system that learns from the feedback obtained through a human-in-the-loop process.

  Architecture available upon request during a potential selection process. 

- **[LazyPwn – Asynchronous CTF Orchestrator](lazypwn.md)** | [🐙 GitHub Repo](https://github.com/marcop-sed/lazypwn)
  Born as a support during HTB Season 9 to automate the classic reconnaissance commands, then gradually expanded. It supports Context-Aware Web Fuzzing (SPA detection and WAF evasion), In-Memory Secret Hunting (JWT and AWS keys extraction from JS sources downloaded on the fly), OSINT via crt.sh, and Auto-Breaching via SSH/SMB cred-spraying if it sniffs passwords. In post-exploitation, a ready arsenal rains down (scripts for Chisel, ELF shells via MSFvenom) and it hands you a convenient interactive report in HTML and via Webhook. It has not been tested outside of those scenarios.

- **[Anime-themed gallery – Boot2Root Design](eth.md)**
  Design of an Ubuntu Server VM to host a controlled vulnerable web app for educational purposes.
  6 vulnerabilities, 3 difficulty levels, progressive exploit chain.


### **Development**

- **Geostment** - *(in progress)*
  A software dedicated to analyzing ETF packages to balance the distribution of investments based on sector and State, to optimize the investment obtaining maximum homogeneity.

- **Game-Engine JS Plugin**
  Plugin to extend the functionalities of a game development engine.



---

## Topics, Technologies and Skills

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
