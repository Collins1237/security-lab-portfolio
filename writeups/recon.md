# Reconnaissance & Host Discovery

**Kill-chain phase:** 1 — Reconnaissance  
**Date:** 2026-10-12  
**Analyst:** Allan Collins  
**Environment:** personal home lab (systems I own / am authorized to test)

> ⚖️ **Scope & authorization.** Every action below was performed only against
> hosts I own or have explicit permission to test. Unauthorized testing of
> third-party systems is illegal (CFAA) — authorization is what separates a
> penetration test from a crime.


## Objective

Enumerate live hosts on the target network and map the attack surface (which machines exist, what they are, and which one is worth attacking next), as the opening phase of a penetration test.


## Tools used

- `ip` — read our own network position
- `nmap -sn` — host discovery (ping/ARP sweep, no port scan)
- `nmap -sV` — service/version detection on the chosen target

## Methodology

**Step 1 — Establish our position.** Identify our own interface, address and default gateway so we know the network we're on.

_Our own IP addresses (ignore loopback)_

```console
$ ip -brief addr | grep -v LOOPBACK
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eno1             UP             192.168.5.131/22 fdcc:584:e235:1:fd3b:2f84:4ddf:482a/64 [global IPv6 redacted]/64 fe80::9fad:40d2:2af6:8354/64 
```

_The default gateway — our router_

```console
$ ip route | awk '/default/{print $3; exit}'
192.168.4.1
```

**Step 2 — Host discovery.** A ping sweep (`nmap -sn`) finds live hosts across the subnet without the noise of a full port scan.

_Ping-sweep the whole LAN_

```console
$ nmap -sn 192.168.4.0/22
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-12 19:05 -0400
Nmap scan report for _gateway (192.168.4.1)
Host is up (0.0099s latency).
Nmap scan report for 192.168.4.21
Host is up (0.049s latency).
Nmap scan report for den.lan (192.168.4.22)
Host is up (0.19s latency).
Nmap scan report for 192.168.4.23
Host is up (2.1s latency).
Nmap scan report for 192.168.4.24
Host is up (0.13s latency).
Nmap scan report for 192.168.4.25
Host is up (2.0s latency).
Nmap scan report for 192.168.4.27
Host is up (0.088s latency).
Nmap scan report for 192.168.4.28
Host is up (0.095s latency).
Nmap scan report for 192.168.4.29
Host is up (0.19s latency).
Nmap scan report for 192.168.4.30
Host is up (0.069s latency).
Nmap scan report for 192.168.4.31
Host is up (0.11s latency).
Nmap scan report for 192.168.4.32
Host is up (0.079s latency).
Nmap scan report for 192.168.4.35
Host is up (0.023s latency).
Nmap scan report for 192.168.4.36
Host is up (0.14s latency).
Nmap scan report for 192.168.4.37
Host is up (0.0070s latency).
Nmap scan report for 192.168.4.38
Host is up (0.73s latency).
Nmap scan report for 192.168.4.39
Host is up (0.0059s latency).
Nmap scan report for 192.168.4.40
Host is up (0.010s latency).
Nmap scan report for 192.168.4.41
Host is up (0.091s latency).
Nmap scan report for 192.168.4.42
Host is up (0.0028s latency).
Nmap scan report for 192.168.4.57
Host is up (0.13s latency).
Nmap scan report for 192.168.4.70
Host is up (1.1s latency).
Nmap scan report for omarchy.local (192.168.5.131)
Host is up (0.000010s latency).
Nmap scan report for 192.168.5.142
Host is up (0.0018s latency).
Nmap done: 1024 IP addresses (24 hosts up) scanned in 44.30 seconds
```

**Step 3 — Service enumeration on the chosen target.** Fingerprint the homelab server (192.168.4.22) with version detection to reveal its exposed services (its attack surface).

_Service + version scan of the target_

```console
$ nmap -sV --version-light 192.168.4.22
Starting Nmap 7.991 ( https://nmap.org ) at 2026-10-12 19:06 -0400
Nmap scan report for den.lan (192.168.4.22)
Host is up (0.089s latency).
Not shown: 995 filtered tcp ports (no-response)
PORT     STATE SERVICE          VERSION
22/tcp   open  ssh              OpenSSH 10.2p1 Ubuntu 2ubuntu3.7 (Ubuntu Linux; protocol 2.0)
53/tcp   open  domain           dnsmasq 2.93 (pi-hole)
80/tcp   open  http?
3001/tcp open  nessus?
8081/tcp open  blackice-icecap?
3 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port80-TCP:V=7.991%I=2%D=10/12%Time=6ACD6801%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,C9,"HTTP/1\.1\x20400\x20Bad\x20Request\r\ncontent-type:\x20app
SF:lication/json\r\nVary:\x20Accept-Encoding\r\nDate:\x20Mon,\x2012\x20Oct
SF:\x202026\x2023:06:41\x20GMT\r\nConnection:\x20close\r\n\r\n{\"error\":\
SF:"Host\x20validation\x20failed\.\x20See\x20logs\x20for\x20more\x20detail
SF:s\.\"}")%r(HTTPOptions,C9,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nconten
SF:t-type:\x20application/json\r\nVary:\x20Accept-Encoding\r\nDate:\x20Mon
SF:,\x2012\x20Oct\x202026\x2023:06:41\x20GMT\r\nConnection:\x20close\r\n\r
SF:\n{\"error\":\"Host\x20validation\x20failed\.\x20See\x20logs\x20for\x20
SF:more\x20details\.\"}")%r(RTSPRequest,C9,"HTTP/1\.1\x20400\x20Bad\x20Req
SF:uest\r\ncontent-type:\x20application/json\r\nVary:\x20Accept-Encoding\r
SF:\nDate:\x20Mon,\x2012\x20Oct\x202026\x2023:06:41\x20GMT\r\nConnection:\
SF:x20close\r\n\r\n{\"error\":\"Host\x20validation\x20failed\.\x20See\x20l
SF:ogs\x20for\x20more\x20details\.\"}")%r(X11Probe,2F,"HTTP/1\.1\x20400\x2
SF:0Bad\x20Request\r\nConnection:\x20close\r\n\r\n")%r(FourOhFourRequest,C
SF:9,"HTTP/1\.1\x20400\x20Bad\x20Request\r\ncontent-type:\x20application/j
SF:son\r\nVary:\x20Accept-Encoding\r\nDate:\x20Mon,\x2012\x20Oct\x202026\x
SF:[global IPv6 redacted]\x20GMT\r\nConnection:\x20close\r\n\r\n{\"error\":\"Host\x20v
SF:alidation\x20failed\.\x20See\x20logs\x20for\x20more\x20details\.\"}")%r
SF:(SSLSessionReq,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x2
SF:0close\r\n\r\n")%r(TLSSessionReq,2F,"HTTP/1\.1\x20400\x20Bad\x20Request
SF:\r\nConnection:\x20close\r\n\r\n");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port3001-TCP:V=7.991%I=2%D=10/12%Time=6ACD67FE%P=x86_64-pc-linux-gnu%r(
SF:NCP,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\
SF:r\n")%r(GetRequest,EC,"HTTP/1\.1\x20302\x20Found\r\nX-Frame-Options:\x2
SF:0SAMEORIGIN\r\nLocation:\x20/dashboard\r\nVary:\x20Accept\r\nContent-Ty
SF:pe:\x20text/plain;\x20charset=utf-8\r\nContent-Length:\x2032\r\nDate:\x
SF:20Mon,\x2012\x20Oct\x202026\x2023:06:44\x20GMT\r\nConnection:\x20close\
SF:r\n\r\nFound\.\x20Redirecting\x20to\x20/dashboard")%r(SSLSessionReq,2F,
SF:"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\r\n")%r
SF:(TLSSessionReq,2F,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x2
SF:0close\r\n\r\n");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port8081-TCP:V=7.991%I=2%D=10/12%Time=6ACD67FE%P=x86_64-pc-linux-gnu%r(
SF:GetRequest,1328,"HTTP/1\.0\x20403\x20Forbidden\r\nCache-Control:\x20no-
SF:cache,\x20no-store,\x20must-revalidate,\x20private,\x20max-age=0\r\nExp
SF:ires:\x200\r\nPragma:\x20no-cache\r\nX-DNS-Prefetch-Control:\x20off\r\n
SF:Content-Security-Policy:\x20default-src\x20'none';\x20connect-src\x20's
SF:elf';\x20font-src\x20'self';\x20frame-ancestors\x20'none';\x20img-src\x
SF:20'self';\x20manifest-src\x20'self';\x20script-src\x20'self';\x20style-
SF:src\x20'self'\x20'unsafe-inline';\x20form-action\x20'self'\r\nX-Frame-O
SF:ptions:\x20DENY\r\nX-XSS-Protection:\x200\r\nX-Content-Type-Options:\x2
SF:0nosniff\r\nReferrer-Policy:\x20strict-origin-when-cross-origin\r\nCont
SF:ent-Type:\x20text/html;\x20charset=utf-8\r\nDate:\x20Mon,\x2012\x20Oct\
SF:x202026\x2023:06:38\x20GMT\r\nConnection:\x20close\r\n\r\n<!doctype\x20
SF:html>\n<!--\n\*\x20\x20Pi-hole:\x20A\x20black\x20hole\x20for\x20Interne
SF:t\x20advertisements\n\*\x20\x20\(c\)\x202017\x20Pi-hole,\x20LLC\x20\(ht
SF:tps://pi-hole\.net\)\n\*\x20\x20Network-wide\x20ad\x20blocking\x20via\x
SF:20your\x20own\x20hardware\.\n\*\n\*\x20\x20This\x20file\x20is\x20copyri
SF:ght\x20under\x20the\x20latest\x20version\x20of\x20the\x20EUPL\.\n\*\x20
SF:\x20Please\x20see\x20LICENSE\x20file\x20for\x20your\x20rights\x20under\
SF:x20this\x20license\.\n-->")%r(FourOhFourRequest,1323,"HTTP/1\.0\x20404\
SF:x20Not\x20Found\r\nCache-Control:\x20no-cache,\x20no-store,\x20must-rev
SF:alidate,\x20private,\x20max-age=0\r\nExpires:\x200\r\nPragma:\x20no-cac
SF:he\r\nX-DNS-Prefetch-Control:\x20off\r\nContent-Security-Policy:\x20def
SF:ault-src\x20'none';\x20connect-src\x20'self';\x20font-src\x20'self';\x2
SF:0frame-ancestors\x20'none';\x20img-src\x20'self';\x20manifest-src\x20's
SF:elf';\x20script-src\x20'self';\x20style-src\x20'self'\x20'unsafe-inline
SF:';\x20form-action\x20'self'\r\nX-Frame-Options:\x20DENY\r\nX-XSS-Protec
SF:tion:\x200\r\nX-Content-Type-Options:\x20nosniff\r\nReferrer-Policy:\x2
SF:0strict-origin-when-cross-origin\r\nContent-Type:\x20text/html;\x20char
SF:set=utf-8\r\nDate:\x20Mon,\x2012\x20Oct\x202026\x2023:06:38\x20GMT\r\nC
SF:onnection:\x20close\r\n\r\n<!doctype\x20html>\n<!--\n\*\x20\x20Pi-hole:
SF:\x20A\x20black\x20hole\x20for\x20Internet\x20advertisements\n\*\x20\x20
SF:\(c\)\x202017\x20Pi-hole,\x20LLC\x20\(https://pi-hole\.net\)\n\*\x20\x2
SF:0Network-wide\x20ad\x20blocking\x20via\x20your\x20own\x20hardware\.\n\*
SF:\n\*\x20\x20This\x20file\x20is\x20copyright\x20under\x20the\x20latest\x
SF:20version\x20of\x20the\x20EUPL\.\n\*\x20\x20Please\x20see\x20LICENSE\x2
SF:0file\x20for\x20your\x20rights\x20under\x20this\x20license\.\n-->");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.92 seconds
```


## Findings

- **24** live hosts discovered on `192.168.4.0/22`.

- Target `192.168.4.22` exposes **5** services (see version scan above). Each open port is a potential entry point and should be justified, patched, and firewalled.

- Mixed environment: servers alongside many IoT devices (see the home network inventory). IoT devices are a common weak link and belong on an isolated segment.


## Remediation / recommendations

- Close or firewall any service that doesn't need to be exposed (reduce attack surface).
- Keep exposed services patched and version-current.
- Segment IoT onto a separate VLAN / guest network so a compromised gadget can't reach servers.
- Monitor for unexpected new hosts or open ports (that's what your Pi-hole/uptime monitoring helps with).

## What I learned (Security+ / pentest concepts)

- Passive vs. active reconnaissance, and why scope/authorization is what makes active recon legal.
- Host discovery (ping sweep) vs. service enumeration (version scan) — two distinct phases.
- Attack surface = the set of reachable services; shrinking it is core defensive hardening.

---
_Generated by the Den pentester training track._
