# Network Switch Hardening — HP/Aruba 2530 from Factory Default

**Topic:** Network device hardening (Security+ Domain 2 · Threats & Vulnerabilities, Domain 4 · Security Operations)  
**Date:** 2026-10-17  
**Author:** Allan Collins  
**Environment:** my own secondhand HP J9774A 2530-8G-PoE+ managed switch, on my home LAN

> Why this matters: network gear is one of the most common soft spots in real
> pentests. Switches and printers often go in with factory settings and nobody
> touches them again. This lab treats my own switch like a target: identify it,
> audit it the way an attacker would, fix every finding, then re-test to prove
> each fix works.

## Objective

Take a secondhand managed switch from out-of-the-box defaults to a hardened
baseline. Verify every change from the network side, not just from the config.

## Recon — identifying the device

I connected the switch's USB console port to my Linux workstation. The kernel picked it up as a USB serial device:

```console
$ journalctl -k | grep -A6 'usb 1-4'
usb 1-4: New USB device found, idVendor=03f0, idProduct=013f
usb 1-4: Product: HPN Serial Port
usb 1-4: Manufacturer: HP
cdc_acm 1-4:1.0: ttyACM0: USB ACM device
```

`HPN` = HP Networking. I opened the console with `picocom -b 115200 /dev/ttyACM0`, which brought up this banner:

```
HP J9774A 2530-8G-PoEP Switch
Software revision YA.15.12.0015
```

## Findings — the default configuration

The console dropped me straight to the admin prompt. **No password.**

```console
HP-2530-8G-PoEP# show running-config
hostname "HP-2530-8G-PoEP"
snmp-server community "public" unrestricted
vlan 1
   name "DEFAULT_VLAN"
   untagged 1-10
   ip address dhcp-bootp
   exit
```

After I plugged it into the LAN, it pulled a DHCP address. I then scanned it from my workstation:

```console
$ sudo nmap -sS -sU -p T:22,23,80,443,U:161 192.168.4.67
22/tcp  open   ssh
23/tcp  open   telnet
80/tcp  open   http
443/tcp closed https
161/udp open   snmp
```

| # | Finding | Risk |
|---|---|---|
| 1 | **No manager password** | Anyone with console or network access has full control |
| 2 | **SNMP community `public` with unrestricted (read-write) access** | Anyone on the LAN can read *and change* the config over SNMP |
| 3 | **Telnet enabled** | Admin credentials cross the network in cleartext |
| 4 | **HTTP web UI enabled (no TLS)** | Same: cleartext credentials |
| 5 | **Firmware from 2014 (YA.15.12)** | ~10 years of missing security fixes; SSH offers only legacy crypto |
| 6 | Flat network: all ports on VLAN 1 | No segmentation (accepted for now in a home lab) |

**Proof of finding 2.** The switch answered an SNMP query using the default community string:

```console
$ sudo nmap -sU -p161 --script snmp-sysdescr --script-args snmpcommunity=public 192.168.4.67
161/udp open  snmp
| snmp-sysdescr: HP J9774A 2530-8G-PoEP Switch, revision YA.15.12.0015, ...
```

**Proof of finding 5.** The SSH service only offers weak algorithms:

```console
$ sudo nmap -p22 --script ssh2-enum-algos 192.168.4.67
|   kex_algorithms:
|       diffie-hellman-group14-sha1
|       diffie-hellman-group1-sha1        <- 1024-bit, considered broken
|   server_host_key_algorithms:
|       ssh-rsa                           <- SHA-1 signatures
|   encryption_algorithms: ... 3des-cbc
|   mac_algorithms: ... hmac-md5
```

## Remediation

All changes were made over the console cable. The console can't be cut off by any network rule, so there was no lockout risk.

```
config
password manager user-name admin          ! finding 1
no snmp-server community public           ! finding 2
no telnet-server                          ! finding 3
no web-management plaintext               ! finding 4
ip authorized-managers 192.168.5.131 255.255.255.255 access manager   ! finding 5 (compensating)
write memory
```

### Lesson learned: verify, don't assume

My first attempt at removing the SNMP community **didn't stick**. The re-scan still got an answer to `public`. A control test with a made-up community string got no answer. That proved the scan was valid and the fix really hadn't applied. I redid the fix, confirmed it with `show snmp-server`, and saved it.

A second gotcha: nmap's `snmp-info` script reported SNMP as "open" and returned
data even after the fix. That script uses SNMPv3 engine discovery, which the switch answers
without any credentials. The real test is whether a *community string* gets accepted.

## Verification (re-scan after hardening)

```console
$ sudo nmap -sS -sU -p T:22,23,80,443,U:161 --script snmp-sysdescr \
      --script-args snmpcommunity=public 192.168.4.67
22/tcp  open   ssh
23/tcp  closed telnet
80/tcp  closed http
443/tcp closed https
161/udp open   snmp        <- listening, but no reply to "public"
```

SSH still works from my authorized workstation. The switch requires a password:

```console
$ ssh -v admin@192.168.4.67 ...
debug1: Remote protocol version 2.0, remote software version Mocana SSH 5.3.1
debug1: Authentications that can continue: password
```

**The allowlist blocks everyone else.** As a negative test, I tried SSH to the
switch from my phone, which has a different IP on the same LAN. The switch **refused the
connection**. Only the authorized workstation can reach the management plane.

| # | Finding | Before | After |
|---|---|---|---|
| 1 | Manager password | none | ✅ set |
| 2 | SNMP `public` read-write | answered | ✅ no response |
| 3 | Telnet (23) | open | ✅ closed |
| 4 | HTTP (80) | open | ✅ closed |
| 5 | Old firmware / weak SSH | exposed to whole LAN | ⚠️ mitigated: management restricted to one host (other devices verified refused) |

## The firmware problem — a compensating control

The real fix for finding 5 is upgrading to YA.16.11. HPE's public release notes
confirm the J9774A is supported on that branch. But the download portal requires
a business email and company name, which ruled me out as a home user.

I found a third-party mirror hosting the image and **chose not to use it**. A
switch sees all traffic that crosses it, and with no vendor checksum to compare against, there's no way to
prove a mirrored image hasn't been tampered with. That's a textbook
supply-chain risk.

Instead I applied a **compensating control**. `ip authorized-managers` restricts
management access to a single workstation IP, which shrinks the exposure of the
weak SSH stack from "anyone on the LAN" to one trusted host. The firmware upgrade stays on
the to-do list until I can get the image from HPE.

## Takeaways

- **Default configs are findings.** Every issue here came from the factory: no password, `public` SNMP, and cleartext management. On a real engagement, this is often how you get in.
- **Re-test every fix.** One of my fixes silently failed, and only the re-scan caught it.
- **Use control tests.** A random community string told me whether my SNMP test was trustworthy.
- **When you can't patch, compensate, and document it.** Restricting management access is a defensible interim answer to "the patch isn't available."
- **Firmware provenance matters.** An unverifiable image from a mirror is a bigger risk than old firmware behind an allowlist.

## Next steps

- Upgrade firmware to YA.16.11 from an official HPE source, then re-run `ssh2-enum-algos` to confirm modern crypto.
- Set the clock via SNTP so logs have real timestamps.
- Use VLANs to segment the lab from the home network.
