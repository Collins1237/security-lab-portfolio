# Linux Server Hardening — Firewall, Kernel & Patching

**Topic:** Host hardening (Security+ Domain 4 · Security Operations)  
**Date:** 2026-10-11  
**Author:** Allan Collins  
**Environment:** my own homelab server — Ubuntu 26.04.1 LTS (`ubuntuserver`, 192.168.4.22)

> Why this matters: a pentester has to know what a hardened host looks like
> to recognise a soft one. This lab hardens a real server I run, audits it
> with Lynis, and verifies every change from the live system.

## Objective

Reduce the attack surface of a self-hosted Linux server with safe,
high-value changes, then prove each one is actually in effect.

## What I did

Two scripts, run with sudo on the server:

1. **`harden.sh`** — host firewall + SSH baseline
   - Installed UFW, allowed OpenSSH **before** enabling it (no lockout), then turned it on.
   - SSH drop-in `/etc/ssh/sshd_config.d/99-harden.conf`: key auth on, root login off.
   - Validated with `sshd -t` before reloading so a typo can't kill SSH.
2. **`fw-harden.sh`** — fixes driven by a Lynis audit
   - **Automatic security updates** (`unattended-upgrades`) — the biggest real-world win.
   - **Kernel network hardening** via `/etc/sysctl.d/99-hardening.conf`.
   - **Disabled rare network protocols** (DCCP, SCTP, RDS, TIPC) — less kernel attack surface.
   - Helper tools: `debsums` (package integrity), `libpam-tmpdir`, `acct` (process accounting).

## Verification (live output from the server)

**Kernel hardening is applied:**

```console
$ sysctl net.ipv4.conf.all.rp_filter net.ipv4.conf.all.accept_redirects \
         net.ipv4.conf.all.send_redirects net.ipv4.conf.all.accept_source_route \
         net.ipv4.conf.all.log_martians kernel.randomize_va_space
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
kernel.randomize_va_space = 2
```

| Setting | What it stops |
|---|---|
| `rp_filter = 1` | Spoofed packets whose source couldn't route back through that interface |
| `accept_redirects = 0` / `send_redirects = 0` | ICMP redirect abuse (traffic hijack / MITM) |
| `accept_source_route = 0` | Attacker-chosen packet routes |
| `log_martians = 1` | Logs impossible-source packets for detection |
| `randomize_va_space = 2` | Full ASLR — makes memory-corruption exploits harder |

**Security updates install themselves:**

```console
$ cat /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
$ systemctl is-active unattended-upgrades
active
```

**Unused protocols can't load:**

```console
$ cat /etc/modprobe.d/99-disable-rare.conf
install dccp /bin/true
install sctp /bin/true
install rds /bin/true
install tipc /bin/true
```

**Firewall is running:**

```console
$ systemctl is-active ufw
active
```

## Finding — Docker ports bypass the host firewall

Auditing the listening ports turned up a gap my own hardening didn't cover:

```console
$ ss -tlnH | awk '{print $4}' | sort -u
0.0.0.0:22      # SSH
0.0.0.0:80      # Caddy (redirects to HTTPS)
0.0.0.0:443     # Caddy HTTPS
0.0.0.0:3001    # Uptime Kuma  — direct, unencrypted
0.0.0.0:8081    # Pi-hole web  — direct, unencrypted
0.0.0.0:9443    # Portainer    — direct
192.168.4.22:53 # Pi-hole DNS
```

Docker writes its own iptables rules for published ports, so they are
reachable from the LAN **even though UFW is enabled** — and they skip the
HTTPS reverse proxy. This is a well-known Docker/UFW pitfall and exactly
the kind of thing an internal pentest flags.

**Remediation (planned):** stop publishing 3001/8081/9443 on the host (or bind
them to `127.0.0.1`) so the only way in is through the HTTPS proxy on 443.

## Takeaways

- Hardening has to be **verified on the live box**, not assumed from the script.
- Patching automatically closes more real-world holes than any clever tweak.
- Containers change the firewall picture — always re-scan after deploying them.
