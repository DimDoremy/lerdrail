# Debugging Loop

## Table of Contents
- [The loop](#the-loop)
- [1. dump / dd](#1-dump--dd)
- [2. Query viewer](#2-query-viewer)
- [3. Profiler (SPX)](#3-profiler-spx)
- [4. Worker health](#4-worker-health)
- [5. Logs](#5-logs)
- [Xdebug](#xdebug)
- [Tinker](#tinker)
- [Pairing with Boost and superpowers](#pairing-with-boost-and-superpowers)

## The loop

Work top-down. Cheap, fast checks first; escalate only when needed.

```
bug / unexpected behavior
   │
   ▼
1. dump/dd      → inspect values at a point (response stays clean)
   │ not enough
   ▼
2. query viewer → N+1 / slow queries in dashboard
   │ not enough
   ▼
3. profiler     → SPX flame graph of the whole request
   │ runtime/worker issue
   ▼
4. worker heal  → failed queue/schedule workers; read worker logs
   │ need raw logs
   ▼
5. logs         → MCP logs tool, grep across app/fpm/worker/nginx
```

Always clean up instruments (`lerd dump off`, `lerd profile off`) before committing — see dev-workflow quality gates.

## 1. dump / dd

`dump()` / `dd()` are the fastest value inspections, but their output is lost when it goes through Blade, a queue worker, or an XHR response. Lerd captures every call and streams it to the dashboard, the System sidebar, the TUI, and MCP — independent of response output.

- Enable: `lerd dump on` (or the antenna button in the Sites sidebar, or MCP `diag` `dumps_toggle`).
- By default the **response stays clean** (dump goes only to the viewer). To also print to the response, set `dumps.passthrough: true` (read at FPM start, so it restarts FPM units).
- CLI: `lerd dump tail [--site X] [--ctx fpm|cli]`
- MCP: `diag` `dumps_recent` / `dumps_status` / `dumps_clear` / `dumps_toggle`
- The same global flag also arms `lerd_devtools` collectors (queries, mail, views, events, jobs, outbound HTTP) in the dashboard's Debug window.
- Mechanism: a `dump-bridge.php` is auto-prepended in every FPM container; toggling only writes/removes a sentinel flag file — **no FPM restart, no worker cascade**.
- 500-event ring buffer in memory; not persisted; loopback only.

**For tests:** `dump($response->json())` inside a test won't pollute PHPUnit output — it lands in the Dumps tab. But remember `lerd dump off` before commit.

## 2. Query viewer

The dashboard's Query tab (and MCP `diag` `analyze_queries`) surfaces executed SQL with bindings, highlighting **N+1 problems** and slow queries. Each Tinker run in Laravel mode also captures its SQL inline (see Tinker). Use it before reaching for the profiler when the symptom is "this page is slow" or "too many queries."

## 3. Profiler (SPX)

For whole-request performance:

```bash
lerd profile on          # arm SPX
# reproduce the request in the browser
lerd profile open        # open the flame graph
lerd profile off         # disarm when done
lerd profile clear       # clear collected data
```

MCP: `diag` `profiler_toggle` / `profiler_status` / `profiler_clear` / `profiler_report`. Also see `diag` `route_timing` and `optimize_route` for route-level timing.

## 4. Worker health

A worker reaches `failed` state via (a) restart-rate-limit cascades (parent FPM restarted too often) or (b) an app crash loop (bad env, broken migration, deleted dependency) exhausting the systemd `StartLimitBurst`.

- Check: `lerd worker list`, or MCP `worker` `health` (returns JSON of unhealthy workers — call this **before** heal).
- Heal: `lerd worker heal` (or MCP `worker` `heal`) clears `failed` state and starts each unit **once**.
- If it re-crashes: the root cause is in the worker logs — `lerd worker logs <name>` or MCP `logs` with `source: worker:<name>`. Heal will not fix a crash loop; fix the underlying error first.

`heal` is consistent across CLI, dashboard, TUI, and MCP — same code path.

## 5. Logs

MCP `logs` tool (best for agents):
1. `action: "sources"` — list all sources: `app:<file>` (framework logs), `fpm`, `worker:<name>`, plus shared infra `nginx`, `dns`, `watcher`, `ui`, `<service>`, `php<ver>`.
2. `action: "fetch"` with `source`, optional `grep` (regex/substring), `since`/`until` (relative like `15m`/`2h30m`, or timestamp), `level` (app logs only), `lines`.
3. The response carries an opaque `cursor`; pass it back as `since` to fetch only newer lines (streaming over request/response).

CLI: `lerd status` gives a health overview; `lerd site:doctor [domain]` does per-site app-level checks (env, key, deps, `composer audit`/`npm audit`, PHP version, slow routes).

## Xdebug

```bash
lerd xdebug on [version] [--mode debug|coverage|develop|profile|trace|gcstats] [--on-demand]
lerd xdebug off
lerd xdebug status
lerd xdebug pause --list / --pid PID     # detach running processes on demand
```

`xdebug.client_host` is auto-probed to `host.containers.internal`; IDE listens on 9003. Use `--on-demand` (`start_with_request=trigger`) for debugging workers/CLI without always-on overhead.

## Tinker

Every site has a **Tinker tab** in the dashboard header — a browser PHP REPL (Monaco editor) with project-aware completion via `phpantom_lsp` (Eloquent models, relations, scopes, casts, facades, vendor classes). Laravel mode runs `php artisan tinker --execute=...`, captures SQL per statement, auto-dumps bare expressions. Shortcuts: `Ctrl/Cmd+Enter` run, `Ctrl+Space` complete. Drafts persist in `localStorage`.

Limits: 30s hard timeout per run, 64KB request body, fresh process per run (no cross-run state). It executes arbitrary PHP = container shell access; only reachable from `127.0.0.1:7073`.

## Pairing with Boost and superpowers

- **Boost** gives app-level root cause that instruments can't: Error Tracking (exceptions with stack), Database Queries (what actually ran), Configuration Access (is the config what you think?). When a runtime symptom is unclear, cross-check with Boost introspection.
- **superpowers:systematic-debugging** is the discipline: don't patch symptoms, find the root cause. Drop → query → profiler → worker-heal → logs is the *instrumentation*; systematic-debugging is the *method* that decides which instrument and interprets results. See `dev-workflow`.
