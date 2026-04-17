# Offensive Security & Engineering Portfolio

Ciao. Sono un Cyber Engineer, o almeno così mi hanno scritto. Ogni tanto sviluppo roba e faccio esercizi di pentest facendo pratica poi sulla parte di reportistica, visto che per le aziende è importante da quanto ho capito.

Qui troverai una raccolta di **assessment simulati** su scenari realistici: AD compromise via MS-SQL, certificate-based escalation, web application injection, cryptographic exploitation. Non sono ingaggi reali (for legal reasons 🙃), ma rendono bene l'idea.

## Perché questo modo di fare reportistica?

Nel penetration testing il 50% è trovare la vulnerabilità. L'altro 50%? Comunicare il danno agli stakeholder, che non è sempre detto capiscano il tecnichese. **Quindi report non=POC tecnico, ma narrativa + findings + remediation** che i team infrastrutturali possono effettivamente attuare. Avrei anche fatto una tesi al riguardo per automatizzare ciò, ma non la pubblico che altrimenti me la fregano; ovviamente se vi interessa ve la giro, ma non si sa mai.

---

## I Report

### 🔴 **Offensive Security Assessments**

- **[Scenario "Imagery"](imagery.md)** – Web Application Pentest (OWASP)  
  Blind XSS → LFI → Command Injection. Come una galleria d'immagini diventa punto di accesso per RCE. CVSS 9.8.

- **[Scenario "Expressway"](expressway.md)** – Network & Cryptography (NIST 800-115)  
  Attacco alla VPN: IKE Phase 1 enumeration → PSK cracking offline → privilege escalation via Sudo CVE. CVSS 9.2.

- **[Scenario "DarkZero"](darkzero.md)** – Active Directory & Infrastructure (OSSTMM)  
  Dominio AD compromise partendo da utente standard: LDAP enumeration → AD CS misconfiguration → ESC1 certificate forgery → Domain Admin. CVSS 9.9.

- **[Scenario "Signed"](signed.md)** – Kerberos Abuse & MS-SQL (PTES)  
  DB esposto → SMB auth coercion → NTLM hash extraction → Silver Ticket forging via PAC manipulation. Due stage di ticket, due livelli di escalation. CVSS 9.8.

### ⚙️ **Projects**

- **[LazyPwn – Asynchronous CTF Orchestrator](lazypwn.md)** – Un framework asincrono e ad eventi scritto in Python 3, progettato per automatizzare la fase di ricognizione durante le CTF. Ottimizza i tempi di scanning, gestisce i crash di rete tramite design stateful e fornisce routine di supporto per il post-exploitation.

- **[Anime-themed gallery – Boot2Root Design](eth.md)** – Progettazione di una VM Ubuntu Server che ospitasse una web app vulnerabile controllata per scopo didattico. 6 vulnerabilità, 3 livelli di difficoltà, exploit chain progressiva.

---

## Argomenti, Tecnologie e Skill

| Area | Metodologie & Strumenti |
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

**Disclaimer:** I report sono sanitizzati – niente credenziali reali, niente IP pubblici compromessi. Tutti i dati provengono da lab isolate (HackTheBox, HTB Season 9, custom VMs).
