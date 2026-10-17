# WireGuard VPN + Network Segmentation

**Topic:** Secure remote access & network segmentation (Security+ Domain 3: Security Architecture)  
**Date:** 2026-10-17  
**Author:** Allan Collins  
**Environment:** my homelab server (Ubuntu Server 26.04, Docker) + main PC on the same LAN  
**Status:** built, verified on the LAN, then **decommissioned** the same day (see [Teardown](#teardown))

> Why this matters: VPNs and segmentation are how real networks keep remote
> users in, attackers out, and a compromised device from reaching everything.
> As a future pentester, knowing how a tunnel is built tells me what an
> attacker on the other end of it can, and can't, touch.

## Objective

Reach my homelab services (dashboard, Pi-hole, monitoring, SSH) securely from
anywhere, **without** giving a VPN device access to the rest of my home
network.

## Why WireGuard (not OpenVPN)

| | WireGuard | OpenVPN |
|---|---|---|
| Config | ~15 lines, one key pair per device | Needs its own CA + certificates |
| Attack surface | ~4,000 lines, in the Linux kernel | Hundreds of thousands of lines |
| Visibility to scanners | Silent: no reply without a valid key | Answers, so it can be fingerprinted |
| Roaming (Wi-Fi ⇄ cellular) | Instant | Often stalls |

OpenVPN still wins on locked-down networks (it can run over TCP 443), and lab
platforms like TryHackMe hand out `.ovpn` files, so I use it as a *client*
on my main PC. For my own server, WireGuard is the better fit.

## Design

```text
 phone / laptop (anywhere)                     homelab server
 10.8.0.2 ──── encrypted UDP 51820 ────▶ wg0 10.8.0.1
                                          │
                                          ├──▶ Caddy :443 → den.lan services   ✅
                                          ├──▶ Pi-hole DNS (ad-blocking)      ✅
                                          ├──▶ SSH (key-only)                 ✅
                                          └──▶ rest of the home LAN           ❌ DROPPED
```

- **Split tunnel:** clients send only the VPN subnet (`10.8.0.0/24`) and the
  homelab server's address through the tunnel. Normal browsing stays on the
  local connection, but DNS goes to Pi-hole, so ad/malware blocking follows
  my phone everywhere.
- **Segmentation enforced on the server**, not just in client configs: a
  firewall rule drops anything arriving on `wg0` that tries to leave toward
  the home LAN. Even if a client edited its own config, the server still
  refuses to route it to other home devices (Home Assistant Pi, router,
  other devices).
- **Least privilege:** each device gets its own key pair and a single `/32`
  address. Revoking one device means deleting one `[Peer]` block.

## Build

All steps are in a re-runnable script (`wg-setup.sh`) run with sudo on the server.

1. **Install** `wireguard` and `qrencode` (QR codes for phone setup).
2. **Keys:** generated on the server, private keys root-only (`600`), never
   copied off the box. My test PC generated its *own* key pair locally and
   only shared the public key.
3. **`/etc/wireguard/wg0.conf`:** the interface `10.8.0.1/24` on UDP 51820,
   one `[Peer]` per device, plus the segmentation rule:
   ```ini
   PostUp   = iptables -I FORWARD 1 -i %i -o wlo1 -j DROP
   PostDown = iptables -D FORWARD -i %i -o wlo1 -j DROP
   ```
4. **Firewall:** `ufw allow 51820/udp` (the only new opening).
5. **Service:** `systemctl enable --now wg-quick@wg0`, so it starts on every boot.
6. **Phone:** a client config rendered as a QR code for the WireGuard app.

## Evidence

Server side (captured by the setup script; keys redacted):

```console
## key files are root-only (private keys never leave this box)
-rw------- root /etc/wireguard/phone.key
-rw------- root /etc/wireguard/server.key

## UFW rule for WireGuard
Status: active
51820/udp                  ALLOW       Anywhere                   # WireGuard

## tunnel interface
wg0              UNKNOWN        10.8.0.1/24

## service enabled + running
enabled
active

## segmentation rule (VPN -> LAN dropped)
-A FORWARD -i wg0 -o wlo1 -j DROP
```

## Verification

I brought up a test peer on my main PC (`10.8.0.3`). To test segmentation, I
deliberately routed one other LAN device (`192.168.4.39`) *into* the tunnel, so
the only thing standing between it and the VPN was the server's firewall rule.

```console
== 1. ping the server through the tunnel
3 packets transmitted, 3 received, 0% packet loss

== 2. SSH through the tunnel
ssh OK: ubuntuserver via 10.8.0.3 59856 10.8.0.1 22

== 3. homelab dashboard (https://den.lan via Caddy) through the tunnel
den.lan -> HTTP 200

== 4. SEGMENTATION: LAN device 192.168.4.39 via tunnel (must FAIL)
3 packets transmitted, 0 received, 100% packet loss
```

Handshake on the test peer (`wg show`, keys redacted):

```console
peer: [redacted]
  endpoint: [server]:51820
  allowed ips: 10.8.0.0/24, 192.168.4.39/32
  latest handshake: 27 seconds ago
  transfer: 62.66 KiB received, 14.45 KiB sent
```

**Result:** the encrypted tunnel works, the homelab's services and SSH are reachable
through it, and the server **blocks** VPN traffic to the rest of the LAN (that
device answers ping normally when reached outside the tunnel).

## What I learned

- **Encryption ≠ access control.** The tunnel protects traffic in transit;
  segmentation decides *what's reachable* once you're inside. Both are needed.
- **Enforce at the chokepoint.** Client-side `AllowedIPs` is a convenience, not
  a control: a user controls their own config. The DROP rule on the server is
  what actually holds.
- **Docker + firewalls interact.** Docker manages its own iptables chains, so I
  inserted my rule at the top of `FORWARD` and verified it with a real test
  instead of assuming UFW alone covered it (an earlier hardening finding: Docker
  port-publishing bypasses UFW).
- **Silent services shrink attack surface:** WireGuard doesn't answer
  unauthenticated packets, so a port scan can't even confirm it's there.

## Teardown

I decommissioned the VPN the same day. I didn't need remote access yet, and an
unused service is unneeded attack surface. Removed: the tunnel and its systemd
service, all keys and configs (`/etc/wireguard`), the `51820/udp` firewall
rule, the status job and the packages. I then confirmed the interface was gone
and the other homelab services were unaffected. The setup scripts are kept so
it can be rebuilt in minutes.

**What I didn't get to:** an off-network test from my phone. It never
connected, for two reasons I diagnosed:

1. **No router port forward yet** for UDP 51820, so the router dropped the
   phone's connection attempts from the internet.
2. **I was testing on my home Wi-Fi.** The phone dialed my public IP from
   inside the house, which doesn't loop back without NAT hairpinning, and at
   home the VPN isn't needed anyway.

**Lesson:** a split tunnel that points DNS through the tunnel breaks *all*
browsing when the tunnel can't connect. Every lookup goes into a dead tunnel.
Next time: set up the port forward, test off-network, and only then hand out
the client config (with on-demand rules so it stays off on the home network).

If I rebuild it: port forward + DHCP reservation, dynamic DNS (my home IP can
change), and wired Ethernet for the server instead of laptop Wi-Fi.

**Security+ tie-ins:** remote-access VPN, split vs. full tunnel, network
segmentation, least privilege, attack-surface reduction, key management.
