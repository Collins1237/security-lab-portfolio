# Home Network Security Assessment — Treating My Own LAN Like a Client

**Topic:** Vulnerability assessment and reporting (Security+ Domain 4 · Security Operations, Domain 2 · Threats & Vulnerabilities)  
**Date:** 2026-10-20  
**Author:** Allan Collins  
**Environment:** my home network. One flat /22 behind a consumer mesh router, a Linux workstation, an Ubuntu homelab server running Pi-hole, and about 40 household devices.

> Why this matters: a pentest gets sold as a **report**, not a scan. This
> assessment runs the whole loop on a network I'm allowed to test: define scope,
> collect evidence, rate the findings, recommend fixes, and track what's still open.
> I wrote it in the same format I'd hand a client.

## Rules of engagement

- **In scope:** every device on my home LAN (192.168.4.0/22), discovered by ARP and DHCP.
- **Authorization:** it's my network, and I own or administer the infrastructure. Devices that belong to other people in the house got the lightest checks (see below).
- **Allowed techniques:** non-intrusive only. TCP connect scans of the top 100 ports, light service fingerprinting, and passive analysis of DNS logs from my own Pi-hole.
- **Not allowed:** exploitation, password guessing, and anything that could crash a device. Any device that belongs to someone else in the house gets a "what is exposed" check only. I don't log in to it or touch its data.
- **Out of scope:** the ISP, the internet side of the router, and cloud accounts.

## Executive summary

The network is in **reasonable shape for a home network**, with one recurring theme: **devices nobody manages.**
The infrastructure I built myself is locked down:
- SSH is key-only on the server and on the lab laptop.
- The managed switch is hardened.
- The workstation firewall blocks unsolicited inbound traffic.

The risk sits in consumer devices that shipped with their own ideas:
- Smart plugs that ignore the network's DNS.
- A printer exposing raw print and management ports to everyone.
- A smart-home hub sharing files over SMB.

| # | Finding | Severity | Status |
|---|---|---|---|
| 1 | Half the devices bypass DNS filtering | Medium | Open, fix planned |
| 2 | Smart-home hub exposes SMB (139/445) to the LAN | Medium | Open, re-verify |
| 3 | Docker publishes server ports past the host firewall | Medium | Open (first found in [hardening](hardening.md)) |
| 4 | Printer accepts unauthenticated raw jobs and exposes its admin UI | Low | Open |
| 5 | Flat network: IoT, personal, and lab devices share one segment | Low | Accepted for now |
| — | Hardened: server SSH, lab laptop SSH, switch, workstation firewall | Info | ✅ |

Severity is my own call, based on likelihood × impact *inside a home LAN*. Each one would be higher on a business network.

## Methodology

### 1. Discovery

Neighbour (ARP) tables and DHCP turned up **41 devices**. I tagged each by MAC vendor (OUI) into infrastructure, servers, personal, media, IoT, and other.

### 2. Port and service scan

The same scan runs automatically every 30 minutes, so findings get re-checked over time instead of being a one-time snapshot:

```console
$ nmap -sT -T4 --top-ports 100 --open -sV --version-light -n -Pn \
       --host-timeout 90s 192.168.4.0/22
```

Result: **40 hosts scanned, 13 answered with open ports, 29 open ports in total.**

### 3. DNS log analysis

Pi-hole is the only DNS server the router hands out, so its query database is a record of every device that uses DNS. I queried it directly:

```console
$ ssh server docker exec pihole pihole-FTL sqlite3 /etc/pihole/pihole-FTL.db \
    "SELECT COUNT(DISTINCT client) FROM queries WHERE timestamp > strftime('%s','now')-86400"
23
```

Then I compared that list against the 41 devices found in step 1.

### Limitation I hit: passive capture on a switched network

I also ran `tshark` on the workstation. But on a switched or Wi-Fi network, a host only sees **its own traffic plus broadcasts**, not other devices' conversations. For whole-house visibility, the Pi-hole logs were the right data source, not packet capture. This is the same reason real assessments ask for a SPAN port or a network tap.

## Findings

### Finding 1 — Half the devices bypass DNS filtering (Medium)

**Evidence.** **21 of 41** devices made *zero* DNS queries to Pi-hole in 24 hours, even though they were online and talking. All **14 of 14** smart plugs and sensors (Espressif and Tuya chipsets) were in that group.

**What it means.** These devices have their DNS server hard-coded (often a public resolver like 8.8.8.8). That has two effects:
- **Blind spot:** I can't see what they look up.
- **No filtering:** blocklists don't apply to them.

A compromised IoT device is one of the classic footholds. On this network it would also be the device I'd have the *least* visibility into.

**Recommendation.**
1. Block outbound DNS (53/tcp, 53/udp, 853/tcp) from everything except the Pi-hole, so hard-coded resolvers fail and the devices fall back to DHCP DNS.
   The current consumer router can't write that rule, so the realistic paths are:
   - Move IoT onto its own network segment (see Finding 5) behind a firewall I control.
   - Or replace the router with one that supports DNS interception.
2. Until then, treat these devices as **untrusted**.

### Finding 2 — Smart-home hub exposes SMB to the LAN (Medium)

**Evidence.** Scan of the smart-home hub:

```
139/tcp  open  netbios-ssn  Samba smbd 4
445/tcp  open  netbios-ssn  Samba smbd 4
80/tcp   open  http         aiohttp
```

**What it means.** SMB is the protocol behind many of the most damaging lateral-movement attacks (EternalBlue/WannaCry). An optional file-sharing add-on is the usual reason a smart-home hub runs it. From any compromised device on the LAN, that share is the next target to try.

**Recommendation.**
- Turn the file-sharing add-on off unless I'm actively using it.
- If I keep it: require a strong password, disable SMBv1 and guest access, and restrict it to the workstation's IP.

**Status.** In the most recent re-scan the hub was offline (all ports filtered, no ping reply). I have to re-verify this once it's back up. *An unreachable host is not a fixed host.*

### Finding 3 — Docker ports bypass the host firewall (Medium)

**Evidence.** On the homelab server, these containers publish on `0.0.0.0`:

```
uptime-kuma  0.0.0.0:3001->3001/tcp
portainer    0.0.0.0:9443->9443/tcp
pihole       0.0.0.0:8081->80/tcp
```

**What it means.** Docker writes its own iptables rules *ahead of* UFW. A "deny" in UFW doesn't stop these ports, so the admin panels are reachable directly from every device on the LAN, not only through the authenticated reverse proxy. Full detail is in my [server hardening writeup](hardening.md).

**Recommendation.** Bind the containers to `127.0.0.1` and reach them only through the Caddy proxy, or add the rules to the `DOCKER-USER` chain.

### Finding 4 — Printer: raw printing and admin UI open to all (Low)

**Evidence.**

```
80/tcp    http      HP DeskJet 4300 series (redirects to HTTPS admin UI)
631/tcp   ipp
8080/tcp  http
9100/tcp  jetdirect
```

**What it means.**
- **Port 9100 (raw/JetDirect)** accepts print jobs and printer-language commands with **no authentication**. Anyone on the LAN can print, and on many models can read or change settings. Printers get overlooked on real engagements all the time.
- **The admin web UI** is reachable from every device.

**Recommendation.**
- Set an admin password on the printer's web UI.
- Disable the protocols that aren't needed (raw 9100 if nothing uses it, plus the extra 8080 listener).
- Keep the firmware current.

### Finding 5 — Flat network (Low, accepted)

Everything shares one /22: IoT, family laptops and phones, the homelab, and my offensive-security lab laptop. Segmentation would contain Findings 1, 2 and 4. I've accepted it for now. VLANs on the hardened switch are the planned fix ([switch writeup](switch.md)).

## What's already hardened

| Asset | Control | Verified by |
|---|---|---|
| Homelab server | SSH key-only, passwords refused | `Permission denied (publickey)` from a client with no key ([ssh](ssh.md)) |
| Lab laptop (Kali) | SSH key-only, the only open port is 22 | Scan shows only `22/tcp OpenSSH 10.4p1` |
| Managed switch | Password, SNMP `public` removed, telnet/HTTP off, management allowlist | Re-scan and negative test ([switch](switch.md)) |
| Workstation | UFW default-deny inbound, every drop logged | Firewall log shows unsolicited inbound being dropped |
| Remote access | WireGuard VPN instead of open ports | ([vpn](vpn.md)) |

## Remediation tracker

| # | Action | Owner | Status |
|---|---|---|---|
| 1 | Separate IoT segment or block outbound DNS except Pi-hole | me | planned |
| 2 | Re-scan hub; disable or lock down SMB | me | to do |
| 3 | Bind Docker admin ports to localhost | me | to do |
| 4 | Printer admin password; disable unused protocols | me | to do |
| 5 | VLANs on the managed switch | me | planned |

I'll update this table, with re-scan output, as each item closes.

## Takeaways

- **The report is the product.** Scope, evidence, severity, and a fix for each finding. A scan with no context is just noise.
- **Pick the right data source.** Packet capture on a switched network missed most of the house. DNS logs showed all of it.
- **Absence of evidence is a finding.** The devices that *never* talked to Pi-hole were more interesting than the ones that did.
- **Offline ≠ fixed.** A host that doesn't answer gets re-tested later, not closed out.
- **Unmanaged devices are the attack surface.** Everything I configured myself held up. Everything that shipped with vendor defaults produced a finding.
