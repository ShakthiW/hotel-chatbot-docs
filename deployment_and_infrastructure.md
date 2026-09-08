# Deployment & Infrastructure

This document covers how `chatbot-demo-api` actually runs in a real (non-local-dev)
environment: the Docker Compose stack, the droplet it's deployed to, environment variables,
migration behavior, and the security-middleware ordering that a deployment needs to get right.
`chatbot-demo-admin` (the Next.js dashboard) is **not** part of this stack — it deploys to
Vercel separately and just points its API base URL at wherever this stack is running.

---

## 1. Topology

```
DigitalOcean Droplet (Ubuntu)
├── docker compose stack
│   ├── postgres     (postgres:16 — self-hosted, NOT the Aiven instance local dev uses)
│   ├── qdrant        (qdrant/qdrant — self-hosted, NOT Qdrant Cloud)
│   └── api           (this repo's Dockerfile — the Go backend)
└── ufw firewall (22, 8080 open)

Vercel
└── chatbot-demo-admin (Next.js) — NEXT_PUBLIC_PROPERTY_API_URL points at the droplet's public IP:8080
```

**Local dev still uses the shared Aiven Postgres and Qdrant Cloud** — this stack is what a real
deployment (or a droplet you're testing against) runs instead. The two environments are not
the same database; don't expect data parity between them.

No reverse proxy / TLS in front of the API yet — it's served directly over plain HTTP on port
8080. This is fine only until real guest traffic depends on it; add a reverse proxy (Caddy is
the simplest option — one `Caddyfile` line gets automatic Let's Encrypt HTTPS) once a domain is
pointed at the droplet.

---

## 2. The Docker Image

`Dockerfile` is a two-stage build:

1. **Builder** (`golang:1.26-alpine`): `go mod download`, then builds two static binaries —
   `/app` (the API server, from `main.go`) and `/app/migrate` (the standalone migration
   command, from `cmd/migrate`, kept in the image as a manual escape hatch even though the API
   server runs migrations itself on boot — see §4).
2. **Runtime** (`gcr.io/distroless/static-debian12:nonroot`): no shell, no package manager,
   just the two binaries and CA certs (needed for outbound TLS to Gemini, WebXPay, and Qdrant
   Cloud if `QDRANT_TLS=true` is ever used against a cloud instance instead of the container in
   this same stack).

The build is pure Go with `CGO_ENABLED=0` — there's no CGO dependency anywhere in `go.mod`
(no sqlite, no cgo-bound libraries), so a static binary is safe and correct here, not a
workaround.

---

## 3. `docker-compose.yml`

Three services on one private Docker network:

| Service | Image | Exposed to host? | Notes |
| :--- | :--- | :--- | :--- |
| `postgres` | `postgres:16` | No | Data persists in the `postgres_data` named volume. `docker compose down -v` destroys it; plain `down`/`up` does not. |
| `qdrant` | `qdrant/qdrant:latest` | No | Data persists in `qdrant_data`. `latest` is used because a specific verified-current stable tag can't be guaranteed accurate from outside a real check against `github.com/qdrant/qdrant/releases` — pin an explicit version before relying on this in a context where an unannounced image update matters. |
| `api` | built from this repo's `Dockerfile` | **Yes**, port 8080 | The only thing that needs to be reachable from the internet — guest widgets and the Vercel-hosted admin both call it directly. |

Postgres and Qdrant are deliberately **not** published to the host — only the `api` container
reaches them, over the compose network's private bridge. Use `docker compose exec postgres
psql -U ...` for direct DB access, or an SSH tunnel for the Qdrant dashboard, rather than
opening either port.

`api` depends on `postgres` with `condition: service_healthy` — not just `service_started` —
because `database.Connect()` calls `log.Fatal` if it can't actually reach Postgres, so racing
the API container's boot against Postgres still initializing would crash-loop it.

---

## 4. Migrations & Seeding

The API server runs GORM `AutoMigrate` **on every boot** by default (`main.go`) — there is no
separate migration step or container in the compose file. This is controlled by two env vars,
both of which are easy to accidentally carry over from a local `.env` where they mean something
different:

- **`SKIP_MIGRATE`** — `true` skips migration entirely. **Do not copy this as `true` from a
  local dev `.env` onto a fresh deployment** — a brand-new database has no schema at all until
  migration runs once. `docker-compose.yml` defaults this to `false` if unset
  (`${SKIP_MIGRATE:-false}`), but an explicit `true` already present in `.env` still wins.
  AutoMigrate is safe to run on every subsequent boot after that — it only adds what's missing,
  never drops or alters destructively.
- **`SEED_DEMO_DATA`** — `true` seeds a demo property, a demo hotel-manager account, and a demo
  platform-admin account **with passwords hardcoded in `migrate/seed_users.go`** (not
  generated, not read from env — literal strings in the source). This is meant for local dev
  convenience only. Leaving it `true` on a real deployment creates real, working logins with
  publicly-readable-if-anyone-has-repo-access passwords. Set it `false` for any deployment that
  isn't purely disposable — and if it was already accidentally left `true` and seeding already
  ran, setting it `false` afterward does **not** retroactively remove what was already created;
  the demo property/accounts need to be deleted or re-passworded directly.

Separately, `SUPERADMIN_EMAIL`/`SUPERADMIN_PASSWORD` seed a **real** founder/developer
super-admin account (`migrate/seed_superadmin.go`) — this one is deliberately independent of
`SEED_DEMO_DATA`, runs unconditionally, and only acts when both env vars are actually set. It
never overwrites an existing account with the same email.

---

## 5. Qdrant: Cloud vs. Self-Hosted

`pkg/knowledge/qdrant.go`'s gRPC client defaults to TLS, because Qdrant Cloud terminates gRPC
over TLS. A self-hosted Qdrant container on the same droplet — reachable only over the private
Docker network — doesn't speak TLS unless separately configured with certs, which is unnecessary
overhead for traffic that never leaves the compose network. Set:

```
QDRANT_TLS=false
```

to use plaintext gRPC against the self-hosted container in this stack. Leave it unset (or
`true`) only if this ever points at Qdrant Cloud instead. The `QDRANT_API_KEY` requirement
applies either way — the self-hosted container in `docker-compose.yml` is configured with
`QDRANT__SERVICE__API_KEY` set to the same value, so even this network-internal instance isn't
running with auth disabled.

---

## 6. Required Environment Variables

See `.env.docker.example` in `chatbot-demo-api/` for the full annotated template with
`openssl rand -hex 32`-style generation guidance for every secret. The three that make the
server refuse to boot at all if unset (`main.go`'s fail-fast checks, not a soft default):

- `JWT_SECRET`
- `ADMIN_SECRET_KEY`
- `INTERNAL_SERVICE_SECRET` — **must match exactly** the same-named env var on the
  `chatbot-demo-admin` (Vercel) side, or every guest-flow server-to-server call from the admin
  app to this API 401s.

---

## 7. Security Middleware Ordering

`pkg/router/router.go` wraps the router in this order (outermost first):
`Recovery → CORS → RateLimit → Logging → mux`.

**CORS must be outermost, wrapping rate limiting — not the other way around.** A rate-limited
request that never reaches the inner CORS layer gets a 429 with no
`Access-Control-Allow-Origin` header at all; a browser treats a cross-origin response missing
that header as a blocked request, not a readable 429. This was a real, live bug (see the
`ui_ux_2.0_phases.md` 2026-09-08 addendum for the full incident) — every rate-limited call from
the admin dashboard surfaced as a bare "Failed to fetch" with the actual 429/`Retry-After`
completely invisible. With CORS outermost, its headers land on the response regardless of what
an inner layer decides, and CORS preflight `OPTIONS` requests get answered by CORS's own early
return before ever reaching the rate limiter — so a preflight itself can never be the thing
that gets throttled and blocks the real request behind it.

`RATE_LIMIT_RPS` (default 10/sec per client IP) and `RATE_LIMIT_BURST` (default 50) are
deliberately generous — this single limiter covers every route, including the staff admin
dashboard, which fires roughly 10 concurrent requests on a single page load plus ongoing
polling. The original default (2 req/sec, burst 20) was tripped by one completely ordinary
dashboard tab, no abuse involved. Set `RATE_LIMIT_RPS=0` to disable entirely for load testing.

---

## 8. Initial Droplet Setup (Ubuntu)

Condensed from an actual first-deployment session. Run as a sudo-capable user (or root, if
you're using DigitalOcean's key-based root login rather than a password — the main risk root
login exists to avoid is already mitigated by key-only auth).

```bash
# 1. If apt is locked by unattended-upgrades on a freshly-booted droplet, wait it out —
#    don't remove the lock file, that can corrupt the package database mid-write.
while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do sleep 5; done

# 2. Docker Engine + Compose plugin (official apt repo, not the convenience script)
apt-get update && apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable docker containerd

# 3. Swap — cheap insurance even on a droplet with headroom (protects against an OOM
#    kill during something heavy, like the Docker build itself)
fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# 4. Firewall
ufw allow OpenSSH
ufw allow 8080/tcp
ufw enable

# 5. Get the repo, configure, deploy
git clone https://github.com/Standord-AI/chatbot-demo-api.git
cd chatbot-demo-api
cp .env.docker.example .env
nano .env   # fill in every value — see §6
docker compose up -d --build

# 6. Verify
docker compose ps
curl http://localhost:8080/health
```

A minimum droplet size of ~2GB RAM is recommended — Postgres + Qdrant + the API together on
anything smaller risks OOM kills under real load even with the swap file above.

---

## 9. Redeploying After a Code Change

```bash
git pull
docker compose up -d --build   # rebuilds only the api image; postgres/qdrant are untouched
```

For a change to `docker-compose.yml` or `.env` itself (not just application code), the same
command picks up the new configuration on the next container recreation.
