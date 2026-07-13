---
name: lerd-dev
description: Use when working on a PHP/Laravel project in the Lerd local dev environment (rootless Podman, *.test domains, MCP tools) alongside Laravel Boost MCP and the superpowers workflow skills. Covers the five-way division of labor (Boost/Lerd/context7/this-harness/superpowers), the 11 grouped Lerd MCP tools, the container-vs-host networking trap (lerd-postgres/lerd-redis vs 127.0.0.1), site link/init/setup/secure workflow, PHP version isolation, and the dump→query→profiler→worker-heal debugging loop.
---

# Lerd Dev Environment

## Overview

Lerd is an open-source, Herd-like local PHP dev environment for Linux/macOS, built on **rootless Podman** (no Docker daemon, no root). It runs Nginx + PHP-FPM + databases (PostgreSQL, Redis/Valkey, Meilisearch, RustFS, Mailpit) as rootless containers, gives every project an automatic `*.test` domain with mkcert HTTPS, and ships an MCP server so AI agents can drive the whole environment from chat.

This project also runs **Laravel Boost MCP** (app introspection), the **superpowers** workflow skills (execution engine), and this **harness skills pack** (conventions + data architecture + workflow discipline). Five distinct AI capability layers cooperate. Knowing which layer owns a task is the single most important thing to get right.

## The Five-Way Division of Labor

Before doing anything, decide which layer owns the task. Read [references/mcp-division.md](references/mcp-division.md) for the full decision table.

| Layer | Owns | Typical tool |
|-------|------|-------------|
| **Boost MCP** | "What does the app look like right now?" — live schema, routes, config, logs, tinker, artisan command list, Laravel doc search | Boost tools (schema/routes/config/logs/tinker/commands/docs) |
| **Lerd MCP / CLI** | "How is the environment running?" — start/stop services, link sites, PHP version, worker health, dump/profiler toggle, DB snapshot, run tests | Lerd CLI (`lerd ...`) or Lerd MCP `exec`/`service`/`worker`/`diag` tools |
| **context7** | Non-Laravel docs — React, Tailwind, Inertia, PostgreSQL, Valkey, ClickHouse, Pest | `mcp__context7__query-docs` |
| **This harness** | "How should the code be written?" — conventions, data architecture, component structure, workflow discipline | The other 4 skills in this pack |
| **superpowers** | "How should work get executed?" — brainstorm, plan, subagent execution, verification, branch finishing | `superpowers:*` skills |

**Call preference:** Laravel app introspection → Boost first. Environment/test/runtime → Lerd. Laravel docs → Boost Documentation Search first, context7 fallback. Everything else non-Laravel → context7.

## The #1 Networking Trap (read this first)

The most frequent, frustrating failure in a Lerd project is a connection refused / SQLSTATE error caused by confusing two different network perspectives:

- **Inside the PHP-FPM container** (where Laravel runs), database/cache hosts are **container hostnames**: `lerd-postgres`, `lerd-redis`, `lerd-mysql`, `lerd-mailpit`, `lerd-meilisearch`, `lerd-rustfs`.
- **On the host** (your laptop, GUI tools, `psql`/`redis-cli` from a terminal), the same services are at `127.0.0.1` on their exposed ports.

`lerd env` writes the correct container hostnames into `.env` for you. **If you ever hand-edit `.env`, never use `127.0.0.1` for `DB_HOST`/`REDIS_HOST`/`MAIL_HOST` — the container cannot reach the host loopback.** Full details and host/port/credential tables in [references/host-networking.md](references/host-networking.md).

## Lerd MCP Tools — 11 grouped tools

Lerd's MCP surface is **11 grouped tools**, each driven by an `action` parameter. Always pass `action`; call the `list`/`health` action first to discover sites or worker state before operating.

`site` · `service` · `db` · `env` · `runtime` · `worker` · `exec` · `framework` · `diag` · `logs` · `worktree`

Quick reference and action tables in [references/mcp-tools.md](references/mcp-tools.md).

> **CLI vs MCP naming:** The Lerd **CLI has no `lerd exec` command.** To run artisan/composer/test from the shell use `lerd artisan`, `lerd test`, `lerd pest`, `lerd npm`, `lerd composer`, `lerd <vendor-bin>`. The word "exec" only appears at the **MCP layer** as the `exec` tool whose actions are `artisan`/`composer`/`console`/`vendor_run` etc.

## Project Workflow

New project: `lerd new <name>` (or `laravel new`) → starter kit prompt asks to add **Boost** → accept → `composer require laravel/boost --dev` + `php artisan boost:install` auto-configures MCP → `lerd link` → `lerd init` (wizard writes `.lerd.yaml`) → `lerd setup` (runs composer install, npm ci, `lerd env`, migrate, build, secure, start workers) → `lerd open`.

Existing project: `cd <project> && lerd link` rebuilds the whole environment from `.lerd.yaml` (including workers, PHP version, services).

Full walkthrough, `.lerd.yaml`, and the Boost + Pest browser-testing install steps in [references/workflow.md](references/workflow.md).

## Debugging Loop

When something breaks, work through this loop rather than guessing:

1. **dump/dd** — `lerd dump on`, then `dump($var)` streams to the dashboard Dumps tab (response stays clean). `lerd dump off` when done.
2. **Query viewer** — catch N+1 and slow queries in the dashboard Query tab.
3. **Profiler (SPX)** — `lerd profile on` → reproduce → `lerd profile open` for a flame graph.
4. **Worker health** — failed queue/schedule workers show a dashboard banner; `lerd worker heal` (or MCP `worker` `heal`) resets and restarts them.
5. **Logs** — MCP `logs` tool: `sources` to list, then `fetch` with `grep`/`since`/`cursor`.

Boost complements this with app-level root cause (Error Tracking, Database Queries). Full loop + wiring in [references/debugging.md](references/debugging.md).

## PHP & Node Versions

Resolution priority (first wins): `.lerd.yaml` `php_version` → `.php-version` file → `composer.json` `require.php` → global default in `~/.config/lerd/config.yaml`.

- Global: `lerd use 8.4`
- Per-project: `lerd isolate 8.4` (writes `.php-version`, syncs `.lerd.yaml`, re-links)
- Node: `lerd isolate:node 20` / `lerd npm` uses the pinned version

## context7 Library IDs (fixed)

Pre-resolved library IDs so you don't re-resolve each session — full query templates in [references/context7-queries.md](references/context7-queries.md).

- Laravel `/laravel/docs` — **Boost Documentation Search first**, context7 fallback
- React `/reactjs/react.dev` · Tailwind `/tailwindlabs/tailwindcss.com`
- Inertia `/inertiajs/docs` + `/inertiajs/inertia-laravel`
- Pest `/pestphp/docs` + `/pestphp/pest-plugin-browser`
- PostgreSQL `/websites/postgresql_current` · Valkey `/websites/valkey_io`
- ClickHouse `/clickhouse/clickhouse-docs`

## Quick Reference

| Want to… | Do |
|----------|----|
| Link current dir as a site | `lerd link` |
| Bootstrap project end-to-end | `lerd setup --all` |
| Run Laravel migrations | `lerd artisan migrate` |
| Run Pest tests | `lerd test` (= `lerd artisan test`) |
| Run a vendor binary | `lerd pest` / `lerd pint` / `lerd phpstan` |
| Build frontend | `lerd npm run build` |
| Open the dashboard | `lerd dashboard` (http://127.0.0.1:7073) |
| See resolved PHP/Node for current site | `lerd which` |
| Heal failed workers | `lerd worker heal` |
| Enable HTTPS | `lerd secure` |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `DB_HOST=127.0.0.1` in `.env` | Use `lerd-postgres` / `lerd-mysql` / `lerd-redis`; re-run `lerd env` |
| Using `lerd exec artisan ...` | No such CLI command — use `lerd artisan ...` |
| Running `php artisan test` from host shell | May hit wrong PHP/version — use `lerd test` |
| Forgetting `lerd dump off` | Leftover `dump()` stays captured; clean before commit |
| Editing DuskTestCase / Selenium manually | Let `lerd env` wire Dusk; for new projects prefer Pest browser testing (see lerdrail-workflow) |
