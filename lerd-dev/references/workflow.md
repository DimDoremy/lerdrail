# Lerd Project Workflow

## Table of Contents
- [New project](#new-project)
- [Existing project](#existing-project)
- [The .lerd.yaml file](#the-lerdyaml-file)
- [Installing Boost](#installing-boost)
- [PHP & Node version management](#php--node-version-management)
- [HTTPS / TLS](#https--tls)
- [Services](#services)
- [Database](#database)
- [Workers](#workers)

## New project

```bash
# Option A: Lerd scaffolds with the framework's create command (default Laravel)
cd ~/Lerd && lerd new myapp

# Option B: use Laravel's installer, then link
laravel new myapp
cd myapp
lerd link
```

During `laravel new` / `lerd new` with a starter kit, you'll be **prompted to add Laravel Boost**. Accept it — this runs `composer require laravel/boost --dev` + `php artisan boost:install` and auto-configures the Boost MCP server. This harness assumes Boost is installed.

Then:
1. `lerd link` — registers the directory as a site, assigns `http://myapp.test` (no `/etc/hosts` edit needed; Lerd's dnsmasq container resolves `.test`).
2. `lerd init` — interactive wizard: pick PHP version, Node version, HTTPS on/off, database, services, which workers to auto-start. Writes `.lerd.yaml`.
3. `lerd setup` — reads `.lerd.yaml` and presents a checkbox list: `composer install`, `npm ci`, `lerd env`, `php artisan migrate`, `storage:link`, `npm run build`, `lerd secure`, `queue:start`, `schedule:start`, `lerd open`. Use `lerd setup --all` to run everything non-interactively (CI-friendly), `--skip-open` to not open the browser.
4. Verify in `lerd status` / `lerd dashboard`: site `active`, services running, workers running.

## Existing project

```bash
cd ~/Lerd/myapp
lerd link
```

`lerd link` rebuilds the entire environment from `.lerd.yaml`: installs dependencies, applies PHP/Node versions, starts the declared services and workers, and configures the vhost. On a fresh machine after `git clone`, `lerd link` (or `lerd setup`) is all you need.

If `~/Lerd` is parked (`lerd park ~/Lerd`), projects under it are auto-linked and you can skip explicit `link`.

## The .lerd.yaml file

`.lerd.yaml` lives at the project root and is **committed to git**. It captures project intent so `lerd link`/`lerd setup` reproduce the environment anywhere. It records (among others):
- `php_version`, Node version
- `secured: true/false`
- services to start
- `workers:` list (queue/schedule/horizon/reverb/...) — every `lerd worker start/stop` updates this; clone-and-link restores them all
- optional custom service definitions and framework commands

Set `mcp_inject: false` if you don't want `lerd update` to refresh injected MCP/skill files in this project.

## Installing Boost

If the starter-kit prompt was skipped or you said no:

```bash
lerd composer require --dev laravel/boost
lerd artisan boost:install
```

`boost:install` registers the Boost MCP server with your AI client. Verify with `lerd artisan boost:status` (or check the client's MCP list). Boost provides live app introspection — see [mcp-division.md](mcp-division.md).

## PHP & Node version management

Supported PHP: 8.5, 8.4, 8.3, 8.2, 8.1 (plus frozen legacy 8.0, 7.4 — opt-in).

Resolution priority (first wins):
1. `.lerd.yaml` `php_version`
2. `.php-version` file
3. `composer.json` `require.php` constraint → best installed version
4. Global default in `~/.config/lerd/config.yaml`

Commands:
- Global default: `lerd use 8.4`
- Pin per project: `lerd isolate 8.4` (writes `.php-version`, syncs `.lerd.yaml`, re-links). Editing `.php-version` directly also works — the watcher updates the vhost.
- List: `lerd php:list`
- Rebuild image: `lerd php:rebuild`
- Extensions: `lerd php:ext add swoole` / `lerd php:ext list`
- Node: `lerd isolate:node 20`; `lerd npm`/`lerd npx` use the pinned version

`lerd install` drops `php` and `composer` shims in `~/.local/share/lerd/bin/` (on PATH) that route to the correct FPM container, so `php artisan ...` and `lerd artisan ...` are equivalent. Prefer `lerd artisan` for clarity.

## HTTPS / TLS

Lerd uses **mkcert** (locally-trusted CA). Requires DNS mode enabled (`dns.enabled: true`); without it, `lerd secure` refuses and sites use `*.localhost`.

```bash
lerd secure          # issue cert for <site>.test + *.<site>.test, regenerate SSL vhost, reload nginx, set APP_URL=https://<site>.test
lerd unsecure
lerd secure --renew  # re-issue on demand
```

Certs live in `~/.local/share/lerd/certs/sites/`. Auto-renewal: certs within ~30 days of expiry are re-issued on the next `lerd start` or watcher pass. Worktrees: securing a parent site auto-enables HTTPS for all worktrees (`*.<site>.test` + `*.branch.<site>.test` SAN). MCP equivalent: `site` `tls_enable`/`tls_disable`/`tls_renew`.

## Services

Built-in services (start with `lerd service start <name>`, MCP `service` `start`):
- PostgreSQL 16 (+ PostGIS 3.5), MySQL 8.4 LTS, Redis 7-alpine, Meilisearch v1.42, RustFS (S3-compatible), Mailpit (SMTP)

Presets (extra services): `lerd service search` / `lerd service preset <name>` (phpMyAdmin, pgAdmin, MongoDB, Selenium, Stripe Mock, …).

Custom services: `lerd service add [file.yaml]` or inline under `services:` in `.lerd.yaml` (OCI image with env injection, placeholders, dependencies, `site_init`).

Update: `lerd service update <name> [tag]`. Cross-version DB migration: `lerd service migrate <name> <target-tag>` (dump+restore), `lerd service rollback`.

## Database

```bash
lerd db:shell                          # interactive psql / mysql
lerd db:export [-d name] [-o file.sql]
lerd db:import [-d name] <file.sql>
lerd db:snapshot [name] [-A]           # named snapshot
lerd db:snapshots                      # list
lerd db:restore <name>
lerd db:snapshot:rm <name>
lerd db:move [--from svc] [--to svc]   # migrate across same-family engines, re-point .env
```

`lerd db:create <name>` creates `<name>` **and** `<name>_testing`. See [host-networking.md](host-networking.md) for the container-vs-host address distinction.

## Workers

Framework workers (queue, schedule, horizon, reverb, stripe, custom) run as systemd-user units (Linux) / launchd jobs (macOS), executing inside the site's PHP-FPM container, auto-restarting on crash.

```bash
lerd queue:start / lerd queue:stop
lerd horizon:start / lerd horizon:stop / lerd horizon:reload
lerd reverb:start / lerd reverb:stop
lerd schedule:start / lerd schedule:stop
lerd worker start/stop/list
lerd worker heal                       # reset-and-start all failed worker units
```

Every start/stop updates `.lerd.yaml`'s `workers:` list → clone-and-link restores them. With `QUEUE_CONNECTION=redis`, Lerd verifies `lerd-redis` is running before starting queue workers. A crash-looping worker is auto-stopped when the site is unlinked.

**Heal semantics:** `lerd worker heal` (or MCP `worker` `heal`) clears the `failed` systemd state and starts the unit once per worker — it does not loop, does not edit `.lerd.yaml`, does not rewrite unit files, and does not touch paused sites. If the worker re-crashes (e.g. bad migration, missing env), fix the root cause via the worker logs (`lerd worker logs <name>` / MCP `logs` `worker:<name>`) before re-healing.
