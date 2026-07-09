# Credential Hunting on Windows

Once you have a foothold on a Windows host, there are several common locations and techniques for finding stored credentials that can help escalate privileges.

## Unattended Windows Installations

When deploying Windows to many hosts, admins often use **Windows Deployment Services**, allowing a single OS image to be pushed to multiple machines over the network. These are called **unattended installations** since they require no user interaction during setup.

Because the initial setup needs an administrator account, credentials may end up stored on disk in files such as:

- `C:\Unattend.xml`
- `C:\Windows\Panther\Unattend.xml`
- `C:\Windows\Panther\Unattend\Unattend.xml`
- `C:\Windows\system32\sysprep.inf`
- `C:\Windows\system32\sysprep\sysprep.xml`

Example of credentials found in these files:

```xml
<Credentials>
    <Username>Administrator</Username>
    <Domain>thm.local</Domain>
    <Password>MyPassword123</Password>
</Credentials>
```

## PowerShell History

Every command run in PowerShell gets saved to a history file — handy for quickly repeating past commands. If a user ever ran a command with a password included inline, it can be recovered later.

From a `cmd.exe` prompt:

```shell-session
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

**Note:** This only works from `cmd.exe`, since PowerShell doesn't recognize `%userprofile%` as an environment variable. From PowerShell, use `$Env:userprofile` instead.

## Saved Windows Credentials

Windows lets you save other users' credentials for reuse. Saved credentials can be listed with:

```shell-session
cmdkey /list
```

You won't see the actual passwords, but if a saved credential looks interesting, you can use it via `runas` with `/savecred`:

```shell-session
runas /savecred /user:admin cmd.exe
```

## IIS Configuration

**Internet Information Services (IIS)** is the default Windows web server. Site configuration is stored in `web.config`, which can contain database or authentication passwords.

Depending on the IIS version, look in:

- `C:\inetpub\wwwroot\web.config`
- `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config`

Quick way to find connection strings:

```shell-session
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config | findstr connectionString
```

## Retrieve Credentials from Software: PuTTY

**PuTTY** is a common SSH client on Windows. It lets users save sessions (IP, username, etc.) for reuse. While PuTTY doesn't store SSH passwords, it *does* store **proxy configuration credentials**, including cleartext passwords.

To search the registry for a saved proxy password:

```shell-session
reg query HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\ /f "Proxy" /s
```

**Note:** "Simon Tatham" is the creator of PuTTY (hence the registry path) — not a username. The associated proxy username should also appear in the output.

### General Principle

Just like PuTTY, **any software that saves passwords** — browsers, email clients, FTP clients, SSH clients, VNC software, etc. — will have some way to recover those saved credentials. These are all worth checking during a credential-hunting phase.
