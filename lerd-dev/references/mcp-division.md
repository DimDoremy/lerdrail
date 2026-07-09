# Five-Way Division of Labor

## Table of Contents
- [Why this matters](#why-this-matters)
- [The decision table](#the-decision-table)
- [Layer details](#layer-details)
- [Call preference rules](#call-preference-rules)
- [Conflict resolution](#conflict-resolution)

## Why this matters

This project runs five distinct AI capability layers that all look superficially similar ("they all help you code"). Picking the wrong layer wastes effort, produces wrong answers, or causes active conflicts — e.g. asking context7 for Laravel docs when Boost has a version-tuned doc API that's more accurate, or hand-inspecting the schema in a SQL file when Boost can introspect the live DB.

Memorize the one-line summaries, then use the decision table.

## The decision table

| Task | Layer | Tool / command |
|------|-------|----------------|
| What columns does the `orders` table have? | **Boost** | `schema` tool |
| List all registered routes | **Boost** | `route inspector` |
| Read a config value (e.g. `queue.default`) | **Boost** | `configuration access` |
| Read the Laravel log | **Boost** | `error tracking` / logs |
| Execute ad-hoc PHP against the app (`User::count()`) | **Boost** | `tinker integration` |
| List available artisan commands | **Boost** | `artisan commands` |
| Look up a Laravel API / how-to | **Boost** doc search → **context7** fallback | Boost `documentation search`, then context7 `/laravel/docs` |
| Run `php artisan migrate` | **Lerd** | `lerd artisan migrate` or MCP `exec` `artisan` |
| Run the test suite | **Lerd** | `lerd test` |
| Start/stop PostgreSQL or Redis | **Lerd** | `lerd service start postgres` / MCP `service` `start` |
| Link the current directory as a site | **Lerd** | `lerd link` / MCP `site` `link` |
| Fix a failed queue worker | **Lerd** | `lerd worker heal` / MCP `worker` `heal` |
| Switch PHP version | **Lerd** | `lerd isolate 8.4` / MCP `runtime` |
| Turn on dump capture / profiler | **Lerd** | `lerd dump on` / `lerd profile on` / MCP `diag` |
| Take a DB snapshot | **Lerd** | `lerd db:snapshot` / MCP `db` `snapshot` |
| Look up React hooks / Tailwind utilities / Inertia forms | **context7** | `query-docs` on the relevant lib |
| Look up PostgreSQL JSONB / Valkey commands / ClickHouse DDL | **context7** | `query-docs` on PG/Valkey/CH |
| How should this model/migration be structured? | **harness** | `laravel-conventions` + `postgres-valkey` skills |
| Is this a normalized entity or an append-only event stream? | **harness** | `postgres-valkey` data-architecture principle |
| How should this Inertia page component be organized? | **harness** | `react-tailwind` skill |
| How do we go from a user request to a committed feature? | **harness** | `dev-workflow` skill |
| TDD red-green cycle, subagent execution, branch finishing | **superpowers** | `superpowers:test-driven-development`, `:subagent-driven-development`, etc. |

## Layer details

### Boost MCP — "what does the app look like right now?"
Laravel's official, app-resident MCP server (`composer require laravel/boost --dev` + `php artisan boost:install`). Provides ~15 tools for **live application introspection**: Application Info, Database Schema, Database Queries, Route Inspector, Artisan Commands, Tinker Integration, Configuration Access, Documentation Search (tuned to the Laravel version actually installed), Error Tracking. Use it whenever you need ground truth about the current app state — it reads the running app, not stale files or pretraining knowledge.

### Lerd MCP / CLI — "how is the environment running?"
Lerd's MCP server (11 grouped tools) and the `lerd` CLI own everything **infrastructural**: site registration (link/park), service lifecycle (start/stop/update databases, caches, mail, search), PHP/Node runtime versions, worker (queue/schedule/horizon/reverb) start/stop/heal, debugging instruments (dump/query/profiler/xdebug toggles), DB create/snapshot/move, TLS. Anything that's "make the environment do X" rather than "read the app's state" is Lerd.

### context7 — non-Laravel documentation
`mcp__context7__query-docs` fetches authoritative, version-current docs for libraries Boost doesn't cover: **React, Tailwind, Inertia.js, PostgreSQL, Valkey, ClickHouse, Pest**. Use it instead of pretraining knowledge for API specifics. Laravel docs are the one exception — Boost's Documentation Search is better there; use context7 only as a fallback.

### This harness — "how should the code be written"
Four sibling skills encoding project conventions and decisions: `laravel-conventions` (models, casts, codegen, testing), `postgres-valkey` (the data-architecture first principle, PG features, Redis/Valkey), `react-tailwind` (Inertia page structure, Tailwind token usage, Vite/SSR), `dev-workflow` (the PRD→plan→execute→accept cycle, test gates, branch discipline). These layers carry **judgment**, which neither Boost nor Lerd nor context7 provide.

### superpowers — "how should work get executed"
The generic execution engine: `brainstorming` (produce a spec/PRD), `writing-plans` (decompose into a TDD plan), `subagent-driven-development` (execute per-task with review), `test-driven-development`, `systematic-debugging`, `verification-before-completion`, `finishing-a-development-branch`, `using-git-worktrees`. `dev-workflow` orchestrates these; you rarely invoke them raw — go through `dev-workflow`.

## Call preference rules

1. **Laravel app introspection → Boost first.** Don't re-derive schema/routes/config from files or memory.
2. **Environment, runtime, test execution → Lerd.** Never `docker`/`composer`/`php` directly.
3. **Laravel docs → Boost Documentation Search first**, context7 `/laravel/docs` fallback.
4. **Non-Laravel docs → context7** (React/Tailwind/Inertia/PG/Valkey/ClickHouse/Pest).
5. **Conventions/decisions/workflow → the harness skills.**
6. **Execution mechanics → superpowers** (via `dev-workflow`).

## Conflict resolution

If two layers seem to apply:
- **Boost vs reading files** for app state → Boost wins (it sees the live app).
- **Boost vs Lerd** for running artisan → either works; prefer Boost if you also need introspection in the same turn, Lerd if it's pure execution.
- **context7 vs pretraining** for any library API → context7 wins (current docs beat stale pretraining).
- **harness vs superpowers** for "how to proceed" → `dev-workflow` decides and delegates to superpowers.
