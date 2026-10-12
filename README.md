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
