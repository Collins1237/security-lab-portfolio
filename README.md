# Security Lab Portfolio

Hands-on penetration-testing practice, documented like real engagements.

I'm working toward a career in penetration testing (currently studying for
CompTIA Security+ SY0-701, Google IT Support certified). This repo is my
homelab practice ground: a set of **self-contained lab exercises I wrote in
Bash**, each one walking through a phase of the penetration-testing kill chain
and auto-producing a **professional writeup** — objective, scope &
authorization, methodology, findings, and remediation.

> ⚖️ **Everything here is authorized and lab-only.** Every exercise targets
> systems I own (my own LAN, my own homelab server) or deliberately-vulnerable
> practice apps I deploy myself (e.g. OWASP Juice Shop) on `localhost`.
> Unauthorized testing of third-party systems is illegal — authorization is
> what separates a penetration test from a crime.

## Writeups

| # | Writeup | Kill-chain phase |
|---|---------|------------------|
| 1 | [Reconnaissance & Host Discovery](writeups/recon.md) | 1 — Reconnaissance |
| 2 | [Password Attacks (Offline Hash Cracking)](writeups/crack.md) | 4 — Password Attacks |
| 3 | [Web Application Security Assessment](writeups/web.md) | 5 — Web App Attacks |

### Defensive groundwork (my homelab server)

Knowing what a hardened host looks like is how you spot a soft one.

| Writeup | Focus |
|---------|-------|
| [Linux Server Hardening](writeups/hardening.md) | UFW, kernel/sysctl hardening, auto-patching, Lynis — plus a Docker-bypasses-firewall **finding** |
| [SSH Key-Only Authentication](writeups/ssh.md) | Staged lockdown, the `sshd_config.d` first-value gotcha, verified from outside |
| [Reverse Proxy + HTTPS (private CA)](writeups/proxy.md) | Caddy, `.lan` TLS via my own certificate authority, trust chains |
| [Version Control with Git](writeups/git.md) | Git fundamentals lab |

## Labs (the automation behind the writeups)

The [`labs/`](labs/) directory holds the exercises themselves — theme-aware,
re-runnable Bash scripts. Each one teaches a concept step by step, runs the
real tools (`nmap`, John the Ripper, Docker), and writes its own report.

| Script | Phase | Target |
|--------|-------|--------|
| [`den-lab-recon`](labs/den-lab-recon) | Reconnaissance | my own LAN |
| [`den-lab-crack`](labs/den-lab-crack) | Password attacks | offline hashes (generated) |
| [`den-lab-web`](labs/den-lab-web) | Web app attacks | OWASP Juice Shop (Docker, localhost) |
| [`den-lab-git`](labs/den-lab-git) | Skills lab | git sandbox |
| [`den-lab-common`](labs/den-lab-common) | — | shared library + writeup engine |

The shared `den-lab-common` library provides the UI helpers and the writeup
engine, so every lab produces a consistent, portfolio-ready Markdown report
(a lab is considered "done" only once its writeup exists — documentation is
the deliverable, exactly like a real engagement).

## Skills demonstrated

- **Recon & enumeration** — host discovery and service/version fingerprinting with `nmap`.
- **Password security** — offline hash cracking, and *why* salting + slow hashes defeat it.
- **Web app security** — the OWASP Top 10, each flaw paired with its remediation.
- **Linux hardening** — firewalling, kernel network hardening, patch automation, auditing with Lynis.
- **Authentication & PKI** — key-only SSH, running a private CA, TLS trust chains.
- **Automation** — reusable Bash tooling with a shared library and a reporting engine.
- **Reporting** — clear writeups with scope, methodology, findings, and fixes.

## Roadmap (pentester training track)

Recon ✅ · Scanning · Vuln Analysis · Passwords ✅ · Web ✅ · Exploitation ·
Privilege Escalation · Post-Exploitation · Reporting

---
*Built in a personal homelab as part of my path from IT support to penetration testing.*
