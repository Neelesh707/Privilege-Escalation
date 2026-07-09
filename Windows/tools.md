# Windows Privilege Escalation — Tools of the Trade

Automated enumeration scripts can speed up the discovery of privilege escalation vectors on a Windows target, but they can also **miss vectors** that manual enumeration would catch. Best practice: use these tools to supplement, not replace, manual enumeration.

---

## WinPEAS

WinPEAS enumerates the target system and prints output covering the same kind of checks done manually (services, permissions, registry, scheduled tasks, etc.).

- Repo / download: https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS
- Available as a precompiled `.exe` or a `.bat` script.

Output can be long — always redirect it to a file:

```cmd
C:\> winpeas.exe > outputfile.txt
```

---

## PrivescCheck

A PowerShell alternative to WinPEAS — no binary execution required, which can help with AV evasion / opsec.

- Repo: https://github.com/itm4n/PrivescCheck

Execution policy may need to be bypassed first:

```powershell
PS C:\> Set-ExecutionPolicy Bypass -Scope process -Force
PS C:\> . .\PrivescCheck.ps1
PS C:\> Invoke-PrivescCheck
```

---

## WES-NG (Windows Exploit Suggester - Next Generation)

Unlike WinPEAS/PrivescCheck, WES-NG runs **on the attacking machine** (e.g., Kali, AttackBox), not on the target. This avoids uploading/executing extra binaries or scripts on the target, reducing the chance of AV detection or leaving noise/artifacts.

- Repo: https://github.com/bitsadmin/wesng
- It checks target patch levels against a local vulnerability database and suggests missing patches with associated exploits.

**Workflow:**

1. Update the local vulnerability database first:
   ```bash
   user@kali$ wes.py --update
   ```

2. On the **target**, run `systeminfo` and redirect output to a file:
   ```cmd
   C:\> systeminfo > systeminfo.txt
   ```

3. Transfer `systeminfo.txt` to the attacking machine.

4. Run WES-NG against it:
   ```bash
   user@kali$ wes.py systeminfo.txt
   ```

---

## Metasploit — `local_exploit_suggester`

If a Meterpreter session is already established on the target, Metasploit can enumerate potential local privilege escalation exploits directly:

```
msf6 > use multi/recon/local_exploit_suggester
msf6 > set SESSION <session_id>
msf6 > run
```

This lists vulnerabilities/modules that may apply to the target and could be used for privilege escalation.

---

## Summary Table

| Tool | Runs On | Type | Notes |
|---|---|---|---|
| WinPEAS | Target | `.exe` / `.bat` | Fast, thorough, but noisy (AV-prone) |
| PrivescCheck | Target | PowerShell script | No binary needed, still runs on target |
| WES-NG | Attacker machine | Python script | Stealthier — only needs `systeminfo` output |
| Metasploit `local_exploit_suggester` | Attacker machine (via active session) | Metasploit module | Requires existing Meterpreter shell |

**Key takeaway:** Automated tools are great time-savers, but don't rely on them exclusively — always cross-check with manual enumeration techniques.
