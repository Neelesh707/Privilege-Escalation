# Linux Privilege Escalation — Finding Sensitive Files (`ls` & `find`)

A quick-reference guide for locating sensitive files, misconfigurations, and privilege escalation vectors using `ls` and `find`.

---

## Checklist

- [ ] `ls -la` in key directories (home dirs, `/tmp`, `/var`, etc.)
- [ ] Search for specific files (`find -name`)
- [ ] Search by file type/permission (SUID, world-writable, world-executable)
- [ ] Search by modification/access time
- [ ] Search by file size
- [ ] Search for dev tools/languages present on the box
- [ ] Wildcard search for likely-sensitive filenames (passwords, keys, etc.)

---

## 1. `ls`

Always use `ls -la` — never just `ls` or `ls -l` — when hunting for privesc vectors. Hidden files (dotfiles) are easy to miss otherwise.

```bash
ls
ls -l
ls -la
```

Example:
```
john@home:~$ ls
john@home:~$ ls -l
total 0
john@home:~$ ls -la
total 24
drwxrwxrwt  4 root  root  4096 Feb 16 04:36 .
drwxr-xr-x 23 root  root  4096 Jun 18  2021 ..
drwxrwxrwt  2 root  root  4096 Feb 16 04:25 .ICE-unix
-rw-rw-r--  1 karen karen   23 Feb 16 04:36 .secret.txt
-r--r--r--  1 root  root    11 Feb 16 04:25 .X0-lock
drwxrwxrwt  2 root  root  4096 Feb 16 04:25 .X11-unix
```

➡️ Plain `ls` and `ls -l` show nothing, but `ls -la` reveals a hidden `.secret.txt` file:

```bash
cat .secret.txt
```
```
This is a secret file.
```

**Takeaway:** Always check for hidden dotfiles — they're a common place to stash notes, credentials, or config files.

---

## 2. `find`

The `find` command is one of the most powerful tools for enumeration — searching the filesystem for interesting files, misconfigured permissions, and privesc vectors.

### Basic file search

```bash
find . -name flag1.txt        # search current dir + subdirs recursively
find /home -name flag1.txt    # search /home + subdirs recursively
```

### Search by type

```bash
find / -type d -name config   # find a directory named "config" under /
```

### Search by permissions

```bash
find / -type f -perm 0777     # files with 777 perms (rwx for everyone)
find / -perm -a=x             # find executable files
```

### Search by owner

```bash
find /home -user frank        # all files owned by user "frank" under /home
```

### Search by time

```bash
find / -mtime -10             # modified in the last 10 days
find / -atime -10             # accessed in the last 10 days
find / -cmin -60              # changed within the last 60 minutes
find / -amin -60              # accessed within the last 60 minutes
```

### Search by size

```bash
find / -size +50M             # files at least 50 MB
find / -size +100M            # files larger than 100 MB
```

`+` and `-` can be used to specify larger/smaller than a given size.

Example:
```
john@home:~$ find / -size +100M
/john/.ZAP/plugin/ZAP_2.15.0_Linux.tar.gz
/john/.gradle/wrapper/dists/gradle-6.8-bin/1jblhjyydfkclfzx1agp92nyl/gradle-6.8-bin.zip
```

⚠️ `find` tends to throw a lot of permission-denied errors that clutter the output. Suppress them with:

```bash
2>/dev/null
```

---

### Finding writable / executable folders

Several equivalent ways to find **world-writable folders**:

```bash
find / -writable -type d 2>/dev/null
find / -perm -222 -type d 2>/dev/null
find / -perm -o w -type d 2>/dev/null
```

Find **world-executable folders**:

```bash
find / -perm -o x -type d 2>/dev/null
```

#### Why three different commands for the same thing? — `-perm` syntax matters

| Syntax | Meaning |
|---|---|
| `-perm mode` | Permission bits must match **exactly** |
| `-perm -mode` | **All** of the specified bits must be set (most common/useful form) |
| `-perm /mode` | **Any** of the specified bits are set |

Example: `-perm g=w` only matches files where group-write is the *only* bit set (mode exactly `0020`). Using `-perm -g=w` instead matches **any** file with group-write permission set — which is almost always what you actually want.

---

### Finding dev tools / scripting languages

Useful for identifying what you could use to write exploits, compile code, or spawn a shell:

```bash
find / -name perl*
find / -name python*
find / -name gcc*
```

---

### Wildcard searches for sensitive files

Use wildcards when you don't know the exact filename you're after — e.g. looking for a password file:

```bash
find / -name pass*.txt
```

This will match `pass.txt`, `password.txt`, `passwords.txt`, etc.

---

### Finding SUID binaries

The SUID bit lets a file run with the privilege level of its **owner**, rather than the user executing it. This is one of the most classic privesc vectors on Linux.

```bash
find / -perm -u=s -type f 2>/dev/null
```

➡️ If a SUID binary owned by root can be abused (e.g., via GTFOBins), it can lead directly to a root shell.

---

## Quick Reference — All Commands

```bash
# ls
ls -la

# basic find
find . -name flag1.txt
find /home -name flag1.txt
find / -type d -name config

# permissions
find / -type f -perm 0777
find / -perm -a=x

# ownership
find /home -user frank

# time-based
find / -mtime -10
find / -atime -10
find / -cmin -60
find / -amin -60

# size-based
find / -size +50M
find / -size +100M

# writable/executable folders
find / -writable -type d 2>/dev/null
find / -perm -222 -type d 2>/dev/null
find / -perm -o w -type d 2>/dev/null
find / -perm -o x -type d 2>/dev/null

# dev tools
find / -name perl*
find / -name python*
find / -name gcc*

# wildcard sensitive file search
find / -name pass*.txt

# SUID binaries
find / -perm -u=s -type f 2>/dev/null
```

---

*Notes based on Linux file-based enumeration fundamentals for privilege escalation (TryHackMe-style walkthrough).*
