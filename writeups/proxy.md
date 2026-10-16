# Reverse Proxy + HTTPS with a Private Certificate Authority

**Topic:** Secure service exposure & PKI (Security+ Domain 1 / Domain 3)  
**Date:** 2026-10-14  
**Author:** Allan Collins  
**Environment:** my homelab server (Docker) + main PC on the same LAN

> Why this matters: every web app a pentester tests sits behind something
> like this. Building it myself means I understand TLS, certificate trust
> chains and what a proxy does — and doesn't — protect.

## Objective

Give every self-hosted service a friendly name and HTTPS, without exposing
anything to the internet, using a private Certificate Authority.

## Design

- **Caddy** runs as a Docker container in front of the other services.
- Local DNS names (`*.den.lan`) come from my Pi-hole.
- `.lan` names can't get public Let's Encrypt certificates, so Caddy acts
  as its **own private CA** (`local_certs`). I trusted its root certificate
  once on each of my machines.

```text
 browser ──HTTPS──▶ Caddy :443 ──▶ homepage:3000      (den.lan)
                              ├──▶ pihole:80         (pihole.den.lan)
                              ├──▶ uptime-kuma:3001  (kuma.den.lan)
                              └──▶ portainer:9443    (portainer.den.lan)
```

## Configuration — `Caddyfile`

```caddyfile
{
	local_certs
}

den.lan {
	reverse_proxy homepage:3000
}

pihole.den.lan {
	redir / /admin/
	reverse_proxy pihole:80
}

kuma.den.lan {
	reverse_proxy uptime-kuma:3001
}

portainer.den.lan {
	# Portainer only speaks HTTPS with its own self-signed cert, inside the
	# private docker network - so skip verifying that one hop.
	reverse_proxy https://portainer:9443 {
		transport http {
			tls_insecure_skip_verify
		}
	}
}
```

The backends are reached by **container name** over the private Docker
network, so they never need their own certificates.

## Verification (from my main PC)

**Every service answers over HTTPS and the certificate validates** (`ssl_verify=0` = trusted, no errors):

```console
$ for h in den.lan pihole.den.lan kuma.den.lan portainer.den.lan; do
>   curl -s -o /dev/null -w "$h %{http_code} ssl_verify=%{ssl_verify_result}\n" https://$h/
> done
den.lan 200 ssl_verify=0
pihole.den.lan 302 ssl_verify=0
kuma.den.lan 302 ssl_verify=0
portainer.den.lan 200 ssl_verify=0
```

**The certificate chains to my private CA, and is short-lived:**

```console
$ openssl s_client -connect den.lan:443 -servername den.lan </dev/null 2>/dev/null \
    | openssl x509 -noout -issuer -dates
issuer=CN=Caddy Local Authority - ECC Intermediate
notBefore=Oct 16 14:11:10 2026 GMT
notAfter=Oct 17 02:11:10 2026 GMT
```

Leaf certificates last ~12 hours and Caddy renews them automatically —
a stolen cert is useless almost immediately.

**Plain HTTP is upgraded to HTTPS:**

```console
$ curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' http://den.lan/
308 https://den.lan/
```

## Trust chain

```text
Caddy Local Authority (root)       ← trusted once on each of my PCs
  └─ Caddy Local Authority - ECC Intermediate
       └─ den.lan (leaf, ~12h, auto-renewed)
```

## Security notes / what I'd fix next

- **`tls_insecure_skip_verify`** on the Portainer hop is a deliberate,
  scoped trade-off: it only skips verification on the internal Docker
  network hop, not for the browser. A cleaner fix is to give Portainer a
  cert from the same private CA.
- **The proxy can be bypassed.** The backend ports (3001, 8081, 9443) are
  still published on the host, so they're reachable directly over the LAN
  without TLS. Next step: stop publishing them so 443 is the only door
  (see my [hardening writeup](hardening.md)).
- **The root CA key is the crown jewel.** Anyone holding it can mint
  trusted certs for my devices — it stays on the server only.
