# Linux Privilege Escalation — Network Enumeration Cheatsheet

A quick-reference guide for enumerating network interfaces and listening ports/connections on a Linux target. Covers `ifconfig` and `netstat`.

---

## Checklist

- [ ] `ifconfig` (or `ip addr`)
- [ ] `netstat -a`
- [ ] `netstat -at` / `netstat -au`
- [ ] `netstat -lt`
- [ ] `netstat -s`
- [ ] `netstat -tp`
- [ ] `netstat -tpln`
- [ ] `netstat -i`
- [ ] `netstat -ano`

---

## 1. `ifconfig`

Shows the system's network interfaces. Useful for identifying pivoting opportunities — e.g., a `docker0` interface suggests Docker is running on the host, which could mean isolated internal networks/containers to pivot into.

```bash
ifconfig
```

Example:
```
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:2aff:feef:cb59  prefixlen 64  scopeid 0x20<link>
        ether 02:42:2a:ef:cb:59  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        TX packets 38  bytes 5128 (5.1 KB)
ens5: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9001
        inet 10.80.96.65  netmask 255.255.192.0  broadcast 10.80.127.255
        inet6 fe80::89f:a8ff:fed4:b8e9  prefixlen 64  scopeid 0x20<link>
        ether 0a:9f:a8:d4:b8:e9  txqueuelen 1000  (Ethernet)
        RX packets 14085  bytes 14572272 (14.5 MB)
        TX packets 4151  bytes 7292416 (7.2 MB)
```

⚠️ **Note:** On modern Linux distros, `ifconfig` has been replaced by the `ip` command. The equivalent is:

```bash
ip addr
```

`ifconfig` still works if the `net-tools` package is installed.

---

## 2. `netstat`

Used to inspect existing network connections, listening ports, and interface statistics.

### `netstat -a`
Shows **all** listening ports and established connections.
```bash
netstat -a
```

### `netstat -at` / `netstat -au`
Limits output to TCP (`-t`) or UDP (`-u`) protocols.
```bash
netstat -at
netstat -au
```

### `netstat -l` (listening ports)
Lists ports in "listening" mode — open and ready to accept connections. Combine with `-t` for TCP only.
```bash
netstat -lt
```
Example:
```
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:ldap            0.0.0.0:*               LISTEN
tcp        0      0 localhost:postgresql    0.0.0.0:*               LISTEN
tcp        0      0 localhost:5433          0.0.0.0:*               LISTEN
```

### `netstat -s` (statistics)
Lists network usage statistics per protocol. Can be combined with `-t` or `-u` to filter.
```bash
netstat -s
```
Example:
```
Ip:
    Forwarding: 1
    66019 total packets received
    5 with invalid addresses
    0 forwarded
    0 incoming packets discarded
    66012 incoming packets delivered
    60939 requests sent out
    74 outgoing packets dropped
Icmp:
    156 ICMP messages received
    0 input ICMP message failed
    ICMP input histogram:
        destination unreachable: 156
    156 ICMP messages sent
    0 ICMP messages failed
    ICMP output histogram:
        destination unreachable: 156
```

### `netstat -tp` (with service + PID)
Lists connections along with the service name and PID info.
```bash
netstat -tp
```
Example:
```
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 10.80.96.65:55010       172.31.66.192:9092      TIME_WAIT   -
tcp        0      0 localhost:5901          localhost:55988         ESTABLISHED
```

### `netstat -tpln` (listening + PID, numeric)
Combine `-l` (listening) with `-t` (TCP), `-p` (PID/program), and `-n` (numeric addresses/ports instead of resolved names).
```bash
netstat -tpln
```

As a normal user, processes owned by other users won't show a PID/program name:
```
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:389            0.0.0.0:*               LISTEN
```

As **root**, the same command reveals the owning process:
```
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:389            0.0.0.0:*               LISTEN      1144/slapd
```

### `netstat -i` (interface statistics)
```bash
netstat -i
```
Example:
```
Kernel Interface table
Iface      MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
docker0   1500        0      0      0 0            72      0      0      0 BMRU
ens5      9001    75667      0      0 0         54412      0      0      0 BMRU
```

### `netstat -ano` (the classic combo)
The most commonly seen netstat usage in write-ups and blog posts:

- `-a` → display all sockets
- `-n` → don't resolve names
- `-o` → display timers

```bash
netstat -ano
```
Example:
```
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       Timer
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 127.0.0.1:5433          0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 0.0.0.0:53              0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 0.0.0.0:5901            0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 0.0.0.0:6001            0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN
```

⚠️ **Note:** On modern Linux distros, `netstat` has been replaced by the `ss` command. Core functionality is similar, with routing info moved to `ip` and some new features added.

Example equivalent:
```bash
ss -tpl        # equivalent of netstat -tpl
```

`netstat` still works if the `net-tools` package is installed.

---

## Quick Reference — All Commands

```bash
ifconfig
ip addr

netstat -a
netstat -at
netstat -au
netstat -lt
netstat -s
netstat -tp
netstat -tpln
netstat -i
netstat -ano

# modern equivalent
ss -tpl
```

---

*Notes based on Linux network enumeration fundamentals for privilege escalation (TryHackMe-style walkthrough).*
