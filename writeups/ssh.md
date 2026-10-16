# SSH Hardening — Key-Only Authentication

**Topic:** Authentication hardening (Security+ Domain 4 · Security Operations)  
**Date:** 2026-10-16  
**Author:** Allan Collins  
**Environment:** my homelab server — Ubuntu 26.04.1 LTS (`ubuntuserver`, 192.168.4.22), managed from my main PC

> Why this matters: SSH with password login is one of the most brute-forced
> services on any network. Key-only auth removes password guessing as an
> attack path entirely — and on the offensive side, it's the first thing I
> check on a Linux target.

## Objective

Lock the server's SSH down to public-key authentication only, with no root
login — without locking myself out — and prove it from the outside.

## Approach — staged, so I could never lock myself out

1. **Stage 1 (2026-10-11, `harden.sh`):** firewall on with SSH allowed first;
   key login on, root login off, **password login deliberately left on** as
   a safety net while I confirmed key login worked every time.
2. **Stage 2 (2026-10-16):** once key login was proven, turned password and
   keyboard-interactive login off.

## The gotcha — sshd uses the *first* value it reads

Ubuntu ships `/etc/ssh/sshd_config.d/50-cloud-init.conf`, and drop-in files
load in alphabetical order. For most options **sshd keeps the first value
it sees**, so a `PasswordAuthentication no` in my `99-harden.conf` could be
silently overridden by an earlier file.

Fix: put the setting in a file that sorts **first**:

```console
$ ls /etc/ssh/sshd_config.d/
00-no-passwords.conf
50-cloud-init.conf
99-harden.conf

$ cat /etc/ssh/sshd_config.d/00-no-passwords.conf
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Applied safely — `sshd -t` validates the config and aborts before reload
if anything is wrong, and my existing session stayed open as a fallback:

```console
$ sudo sshd -t && sudo systemctl reload ssh
```

Lesson: **always check the effective config (`sshd -T`), not just the file
you edited.**

## Verification — tested from my main PC

**Key login still works:**

```console
$ ssh server 'echo key login OK'
key login OK
```

**Password login is refused** (forcing password auth only):

```console
$ ssh -o PubkeyAuthentication=no \
      -o PreferredAuthentications=password,keyboard-interactive server
allan@192.168.4.22: Permission denied (publickey).
```

**Root login is refused:**

```console
$ ssh root@192.168.4.22
root@192.168.4.22: Permission denied (publickey).
```

**The server only offers public-key auth:**

```console
$ ssh -v -o PubkeyAuthentication=no server 2>&1 | grep 'can continue'
debug1: Authentications that can continue: publickey
```

## Result

| Control | Before | After |
|---|---|---|
| Password login | allowed (safety net) | **disabled** |
| Keyboard-interactive | allowed | **disabled** |
| Root login | disabled | disabled |
| Public-key login | enabled | enabled — only method offered |
| Host firewall (UFW) | on | on |

## Attacker's view

A tool like Hydra pointed at this server now gets nothing to guess at — the
server never offers a password prompt. The remaining risk moves to the
private key itself, so the next step is protecting it (passphrase + agent)
and optionally adding TOTP MFA.
