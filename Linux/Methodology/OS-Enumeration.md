# Linux Privilege Escalation — OS Enumeration Cheatsheet

A quick-reference checklist for enumerating operating system properties on a Linux target during privilege escalation. Covers `hostname`, `uname`, `/proc/version`, `/etc/issue`, `ps`, cron jobs, and `dpkg`.

---

## Checklist

- [ ] `hostname`
- [ ] `uname -a`
- [ ] `cat /proc/version`
- [ ] `cat /etc/issue`
- [ ] `ps aux` / `ps axjf`
- [ ] Cron jobs (`/etc/crontab`, `/var/spool/cron/`, `/etc/cron.d/`)
- [ ] `dpkg -l`

---

## 1. `hostname`

Returns the hostname of the target machine.

```bash
hostname
```

- Can be a meaningless string (e.g. `Ubuntu-3487340239`)
- Sometimes reveals the system's role in the network (e.g. `SQL-PROD-01` → production SQL server)

---

## 2. `uname -a`

Prints system/kernel information — useful for spotting kernel exploits.

```bash
uname -a
```

Example:
```
Linux home 6.8.0-41-generic
```

| Value | Meaning |
|---|---|
| `Linux` | Operating system |
| `home` | Hostname |
| `6.8.0-41-generic` | Kernel version |

---

## 3. `/proc/version`

The `procfs` filesystem exposes process/system info. Present on most Linux distros.

```bash
cat /proc/version
```

Example:
```
Linux version 6.8.0-41-generic (buildd@lcy02-amd64-077) (x86_64-linux-gnu-gcc-13 (Ubuntu 13.2.0-23ubuntu4) 13.2.0, GNU ld (GNU Binutils for Ubuntu) 2.42)
```

| Value | Meaning |
|---|---|
| `6.8.0-41-generic` | Kernel version |
| `buildd@lcy02-amd64-077` | Build machine that compiled the kernel |
| `x86_64-linux-gnu-gcc-13` | Compiler used to build the kernel |
| `Ubuntu 13.2.0-23ubuntu4` | Ubuntu package version of gcc |
| `GNU ld ... 2.42` | Linker used during kernel build |

---

## 4. `/etc/issue`

Contains OS/distro identification info.

```bash
cat /etc/issue
```

Example:
```
Ubuntu 22.04.1 LTS \n \l
```

⚠️ Like any system info file, this can be edited/customized — cross-check against other sources (`uname`, `/proc/version`, etc.) for accuracy.

---

## 5. `ps` — Process Status

Shows running processes.

| Field | Meaning |
|---|---|
| PID | Process ID (unique) |
| TTY | Terminal type used by the user |
| TIME | CPU time used (NOT total runtime) |
| CMD | Command/executable (no arguments shown) |

### Useful options

**`ps aux`**
- `a` — all users' processes
- `u` — show the user that launched the process
- `x` — include processes not attached to a terminal

**`ps axjf`**
- `a` — all users
- `x` — no controlling terminal
- `j` — jobs format
- `f` — forest/tree view

Example:
```bash
ps axjf
```
```
   1  1039  1039  1039 ?           -1 Ss       0   0:00 /usr/sbin/sshd -D
1039  1854  1854  1854 ?           -1 Ss       0   0:00  \_ sshd: karen [priv]
1854  1890  1854  1854 ?           -1 S     1001   0:00      \_ sshd: karen@pts/0
1890  1891  1891  1891 pts/0     1975 Ss    1001   0:00          \_ -sh
1891  1975  1975  1891 pts/0     1975 R+    1001   0:00              \_ ps axjf
```

---

## 6. Cron Jobs

Cron = time-based job scheduler. Jobs sometimes run as privileged users — a great privesc target if you can modify a script a cron job calls.

### Where to check
```bash
cat /etc/crontab
ls -la /var/spool/cron/
ls -la /etc/cron.d/
```

### Crontab field breakdown

```
30 2 * * 1 root /home/ubuntu/clear-mail.sh
```

| Field | Value | Range | Meaning |
|---|---|---|---|
| 1 | `30` | 0-59 | Minute |
| 2 | `2` | 0-23 | Hour |
| 3 | `*` | 1-31 | Day of month |
| 4 | `*` | 1-12 | Month |
| 5 | `1` | 0-7 (0 & 7 = Sunday) | Day of week |
| 6 | `root` | — | User that runs the task |
| 7 | `/home/ubuntu/clear-mail.sh` | — | Command/script to run |

➡️ This entry runs `/home/ubuntu/clear-mail.sh` **every Monday at 2:30 AM as root**.

**Privesc angle:** If a root-owned cron job executes a script that's writable by your current user, you can inject a payload and escalate on the next run.

---

## 7. `dpkg -l`

Lists all installed packages and their versions.

```bash
dpkg -l
```

- Useful for identifying outdated/vulnerable software with known public exploits.

