# Host Networking — Container vs Host

## Table of Contents
- [The core rule](#the-core-rule)
- [Why this happens](#why-this-happens)
- [Service host/port/credential table](#service-hostportcredential-table)
- [How to get the right values](#how-to-get-the-right-values)
- [Common symptoms](#common-symptoms)
- [Testing database](#testing-database)

## The core rule

There are **two network perspectives** in a Lerd project, and confusing them is the #1 source of connection errors:

| Perspective | Where it runs | DB/cache host to use |
|-------------|---------------|----------------------|
| **The Laravel app** | Inside the PHP-FPM container (on the `lerd` Podman network) | **Container hostname** — `lerd-postgres`, `lerd-redis`, `lerd-mysql`, `lerd-mailpit`, `lerd-meilisearch`, `lerd-rustfs` |
| **Host tools** (your laptop: GUI DB clients, `psql`, `redis-cli`, TablePlus, DBeaver) | On the host OS | **`127.0.0.1`** on the exposed port |

**Never put `127.0.0.1` in `.env` for `DB_HOST` / `REDIS_HOST` / `MAIL_HOST` / search host.** The FPM container's loopback is its own, not the host's.

## Why this happens

PHP-FPM, Nginx, and all services run as containers on a shared Podman network named `lerd`. Containers address each other by name (Podman DNS). The host reaches services via ports Lerd publishes to `127.0.0.1`. So:
- `lerd-postgres:5432` is how the FPM container connects (same network).
- `127.0.0.1:5432` is how a host tool connects (published port).

Both reach the same database. The difference is *who is asking*.

## Service host/port/credential table

Same DB, two addresses. Credentials are identical either way.

| Service | Container host (`.env`) | Host address (GUI tools) | Port | User | Password | Default DB |
|---------|------------------------|--------------------------|------|------|----------|-----------|
| PostgreSQL 16 (+PostGIS) | `lerd-postgres` | `127.0.0.1` | 5432 | `postgres` | `lerd` | `lerd` (or per-site) |
| MySQL 8.4 | `lerd-mysql` | `127.0.0.1` | 3306 | `root` | `lerd` | `lerd` |
| Redis 7 / Valkey | `lerd-redis` | `127.0.0.1` | 6379 | — | — | — |
| Meilisearch | `lerd-meilisearch` | `127.0.0.1` | 7700 | — | — | — |
| RustFS (S3) | `lerd-rustfs` | `127.0.0.1` | 9000 | `lerd` | `lerdpassword` | per-site bucket |
| Mailpit (SMTP) | `lerd-mailpit` | `127.0.0.1` | 1025 | — | — | — |

Web UIs (always host-side, `127.0.0.1`):
- Lerd dashboard: `http://127.0.0.1:7073`
- Mailpit: `http://127.0.0.1:8025`
- RustFS console: `http://127.0.0.1:9001`

## How to get the right values

**`lerd env`** is the source of truth. When you start a service, Lerd prints the exact `.env` lines to paste, and `lerd env` writes them into `.env` (backing up to `.env.before_lerd` first). It sets, among others:

```
DB_HOST=lerd-postgres        # NOT 127.0.0.1
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=lerd
REDIS_HOST=lerd-redis        # NOT 127.0.0.1
MAIL_HOST=lerd-mailpit
```

If you've hand-edited `.env` and broken it, the fastest fix is `lerd env` to rewrite the correct container hostnames, or `lerd env:restore` to roll back to `.env.before_lerd`.

## Common symptoms

| Symptom | Cause | Fix |
|---------|-------|-----|
| `SQLSTATE[08006] ... Connection refused` from the app | `DB_HOST=127.0.0.1` in `.env` | `lerd env` (writes `lerd-postgres`) |
| `Connection refused` from `psql` on host | Used `lerd-postgres` in a host tool | Use `127.0.0.1` for host tools |
| Redis `Connection refused` in app | `REDIS_HOST=127.0.0.1`, or `lerd-redis` not started | `lerd env`; `lerd service start redis` |
| Queue worker won't start with redis connection error | `lerd-redis` not running | Lerd checks and prompts — start redis first |
| SSL cert error from host tool | Tool not trusting mkcert root | Use `127.0.0.1` (not `.test`) for raw DB connections; for HTTPS to `.test` the mkcert CA must be installed (Lerd does this at `lerd install`) |

## Testing database

`lerd db:create <name>` (and `lerd setup`) always creates **two** databases: `<name>` and `<name>_testing`, with identical credentials. The standard Laravel `phpunit.xml` overrides `DB_DATABASE=testing` for the test environment, so tests hit `<name>_testing` automatically. You do not need to edit `phpunit.xml`.

See `lerdrail-workflow`'s testing gates for how tests run under Lerd.
