# app-loader

A dumb edge: it splits incoming traffic by hostname and forwards it to whichever app is
running. It terminates no TLS.

- **:443** — L4, TLS passthrough by SNI. The apps terminate their own TLS.
- **:80** — L7, Host routing, only so the apps' Caddies can solve ACME HTTP-01 challenges.

Apps address nothing here; this config addresses the apps. Each app publishes itself on a
unique loopback port, so app internals (service names, networks) are irrelevant.

## Hosts

| Host | Backend | HTTPS port | HTTP port |
| --- | --- | --- | --- |
| polysim.work | home | 8446 | 8083 |
| yawa.polysim.work / auth.yawa.polysim.work | yawa | 8443 | 8080 |
| app.polysim.work / auth.app.polysim.work | polysim | 8444 | 8081 |
| enterprise-engine.polysim.work / auth.enterprise-engine.polysim.work | enterprise-engine | 8445 | 8082 |

## Run

```sh
docker compose up -d
```

## Add an app

1. Give it a free port pair, e.g. `9000` (HTTP) and `9001` (HTTPS).
2. In the app's `.env`:
   ```env
   CADDY_HTTP_BIND=127.0.0.1:9000:80
   CADDY_HTTPS_BIND=127.0.0.1:9001:443
   ```
3. Add two `use_backend` lines (443, by SNI) and two (80, by Host) here, plus the matching
   `backend` blocks.
4. Add the DNS records for `<app>.polysim.work` and `auth.<app>.polysim.work`.

Standalone, an app leaves `CADDY_*_BIND` unset and binds 80/443 itself.

## Production notes

### Binding the privileged ports

`haproxy:3.4-alpine` defaults to the unprivileged user `haproxy` (uid 99). On a host with the
usual `net.ipv4.ip_unprivileged_port_start=1024` that user cannot bind `:80`/`:443`, so the
compose file runs the container as `root`; `haproxy` then drops its workers to the `haproxy`
user (see the `global` section of `haproxy.cfg`).

Alternatives, if you never want the container to run as root:

- build a small image that sets `cap_net_bind_service+ep` on the `haproxy` binary, or
- lower the host's `net.ipv4.ip_unprivileged_port_start` (host-wide; affects every user).

Do **not** rely on `cap_add: NET_BIND_SERVICE` — Docker does not grant effective capabilities
to a non-root container user.

### Cloudflare

The edge is pure TLS passthrough, so the Cloudflare proxy mode per record matters:

- **DNS-only (grey cloud)** — clients connect straight to this host; haproxy sees the real
  SNI and each app terminates TLS with its own Let's Encrypt certificate. Simplest and the
  most robust for ACME. Recommended while certificates are issued.
- **Proxied (orange cloud)** — Cloudflare terminates the client TLS and re-originates to the
  edge. This only works with SSL/TLS mode **Full (strict)** (Cloudflare validates the app's
  certificate against the hostname). Caveats:
  - Caddy's preferred `tls-alpn-01` challenge never reaches the origin through Cloudflare, so
    issuance falls back to `http-01` via the edge's `:80`; Cloudflare's *Always Use HTTPS* can
    interfere. Issue certificates with the records **grey**, then switch to **orange**.
  - The robust way to run ACME behind Cloudflare is the **DNS-01** challenge, e.g.
    `tls { dns cloudflare {env.CLOUDFLARE_API_TOKEN} }` in each app's Caddyfile.
