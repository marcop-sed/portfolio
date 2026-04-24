# Offensive Security & Engineering Portfolio

Sono un Cyber Engineer con conoscenze orizzontali con l'obbiettivo di verticalizzarsi maggiormente su DevSecOps e Offensive Security. Ho creato questo portfolio per tenere traccia dei miei lavori.

Attualmente, il menù offre:

- **Assessment simulati**: Report in stile professionale su scenari di HackTheBox, utilizzati per apprendere le metologie e prendere familiarità con gli standard utilizzati. AD compromise via MS-SQL, certificate-based escalation, web application injection, sono alcuni esempi di quello che è stato affrontato. Non sono ingaggi reali (for legal reasons 🙃), ma rendono bene l'idea e sono utili allo scopo.

- **Progetti personali**: Progetti di sviluppo sia in ambito cybersecurity che non. La maggior parte di questi nascono da delle esigenze, per semplificare ed automatizzare processi.



### Perché fare Report su macchine di HTB

Stavo partecipando alla Season 9 di HackTheBox per imparare e sperimentare tecniche di Penetration Testing; in seguito, mi è stato detto che potevano essere un'ottima simulazione per approfondire le tecniche di reportistica, apprendendo le metodologie relative ai framework NIST 800-115, OSSTMM e PTES. 

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


## ⚙️ **Projects**

### **Cybersecurity**

- **NERVA: Nexus for Enterprise Risk and Vulnerability Assessment** - *Master's Thesis Project*  
  NERVA nasce a seguito di un tirocinio in ambito Purple Team per ricerca. L'idea è stata quella di creare un tool che prendesse in input un report del Red Team e, tramite un motore di correlazione che si aggancia ad un LLM, mappare le vulnerabilità sul framework NIST CSF 2.0 e sulla matrice del MITRE ATT&CK; utilizzando quest'ultimo, si identifica la rispettiva remediation proposta dal D3FEND e si propone una patch applicabile, ottenuta e rifinita da un sistema di learning interno che apprende dai feedback ottenuti tramite un processo human-in-the-loop.  
  Architettura disponibile su richiesta durante un eventuale iter di selezione.

- **[LazyPwn – Asynchronous CTF Orchestrator](lazypwn.md)** | [🐙 GitHub Repo](https://github.com/marcop-sed/lazypwn)  
  Nasce come un supporto nato durante la Season 9 di HTB per automatizzare i classici comandi dei reconnaissance, poi man mano ampliato. Supporta Context-Aware Web Fuzzing (rilevamento SPA e WAF evasion), In-Memory Secret Hunting (estrazione JWT e chiavi AWS dai sorgenti JS scaricati al volo), OSINT via crt.sh e Auto-Breaching via cred-spraying SSH/SMB se sniffa password. In post-exploitation piove un arsenale pronto (script per Chisel, shell ELF via MSFvenom) e ti rifila un comodo report interattivo in HTML e via Webhook. Non è stato testato al di fuori di quegli scenari.

- **[Anime-themed gallery – Boot2Root Design](eth.md)**  
  Progettazione di una VM Ubuntu Server che ospitasse una web app vulnerabile controllata per scopo didattico. 
  6 vulnerabilità, 3 livelli di difficoltà, exploit chain progressiva.


### **Development**
 
- **Geostment** - *(in progress)*  
  Un software dedicato all'analisi di pacchetti ETF per bilanciare la distribuzione degli investimenti in base ad ambito e Stato, per ottimizzare l'investimento ottenendo la massima omogeneità.

- **Game-Engine JS Plugin**  
  Plugin per estendere funzionalità di un motore di sviluppo di gioco.

 

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

---
