# Linux Privilege Escalation — User Enumeration Cheatsheet

A quick-reference checklist for enumerating local users on a Linux target during privilege escalation. Covers `id`, `env`, `history`, `sudo -l`, and `/etc/passwd`.

---

## Checklist

- [ ] `id`
- [ ] `env`
- [ ] `history`
- [ ] `sudo -l`
- [ ] `cat /etc/passwd`

---

## 1. `id`

Shows the current user's privilege level and group memberships.

```bash
id
```

You can also check another user's info:

```bash
id matt
```

Example:
```
john@home:~$ id
uid=1001(john) gid=1001(john) groups=1001(john),100(users)
john@home:~$ id matt
uid=1002(matt) gid=1002(matt) groups=1002(matt),27(sudo),116(admin)
```

➡️ Note `matt` is in the `sudo` and `admin` groups — a strong privesc lead if you can access his account.

---

## 2. `env`

Displays environment variables for the current session.

```bash
env
```

Example:
```
MAIL=/var/mail/john
USER=john
SSH_CLIENT=10.80.88.217 50500 22
HOME=/home/john
SSH_TTY=/dev/pts/0
QT_QPA_PLATFORMTHEME=appmenu-qt5
LOGNAME=john
TERM=linux
```

**Privesc angle:** Check the `PATH` variable — if it includes a writable directory, or points to a compiler/scripting language (e.g. Python, Perl), it could be leveraged to run arbitrary code or escalate privileges.

---

## 3. `history`

Shows previously run commands for the current user.

```bash
history
```

- Can reveal target system usage patterns
- Occasionally leaks sensitive info like passwords or usernames typed on the command line

---

## 4. `sudo -l`

Lists commands the current user is permitted to run with `sudo`.

```bash
sudo -l
```

- May require the current user's password
- If specific binaries are allowed as root, check [GTFOBins](https://gtfobins.github.io/) for known privesc techniques for that binary

---

## 5. `/etc/passwd`

A readable file that lists all system accounts — an easy way to enumerate users.

```bash
cat /etc/passwd
```

Example:
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
```

### Extracting just usernames

```bash
cat /etc/passwd | cut -d ":" -f 1
```

```
root
daemon
bin
sys
sync
games
man
lp
mail
news
uucp
proxy
www-data
backup
list
irc
gnats
nobody
libuuid
```

⚠️ This includes system/service accounts that aren't useful for brute-forcing or lateral movement.

### Filtering for real (human) users

Real users typically have a home directory under `/home`, so grep for that:

```bash
cat /etc/passwd | grep /home
```

```
matt:x:1000:1000:matt,,,:/home/matt:/bin/bash
karen:x:1001:1001::/home/karen:
```

➡️ This gives a much cleaner, targeted list of actual human accounts on the system — useful for brute-force attacks or identifying escalation targets.

---
