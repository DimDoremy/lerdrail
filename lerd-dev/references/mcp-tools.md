# Lerd MCP Tools — 11 grouped tools

## Table of Contents
- [How grouped tools work](#how-grouped-tools-work)
- [Tool action reference](#tool-action-reference)
- [Path/site resolution](#pathsite-resolution)
- [Agent variable passthrough](#agent-variable-passthrough)

## How grouped tools work

Lerd's MCP surface is **11 grouped tools**. Each tool takes an `action` parameter that selects the operation — earlier Lerd exposed ~80 flat tools; they were merged to cut token cost and improve tool selection accuracy.

Rules:
- **Always pass `action`.** A call without it does nothing.
- **List/health first.** Before `site` operations call `site` `list` to discover the site; before `worker` operations call `worker` `health` (or `list`) to see state.
- **CLI ↔ MCP mapping.** The MCP `exec` tool's `artisan`/`composer`/`console` actions correspond to the `lerd artisan` / `lerd composer` / `lerd console` CLI commands. There is **no `lerd exec` CLI command**.

## Tool action reference

| Tool | Key actions |
|------|-------------|
| `site` | `list`, `link`, `unlink`, `domain_add`/`domain_remove`, `group_assign`/`group_unassign`/`group_label`/`group_db`/`group_list`, `tls_enable`/`tls_disable`/`tls_renew`, `php`, `node`, `pause`/`unpause`, `restart`, `rebuild`, `runtime`, `nginx_read`/`nginx_write`/`nginx_reset`, `park`/`unpark` |
| `service` | `start`, `stop`, `restart`, `pin`/`unpin`, `update`, `rollback`, `migrate`, `remove`, `reinstall`, `add`, `expose`, `port`, `env`, `config_read`/`config_write`/`config_restore`/`config_reset`/`config_list_backups`, `preset_list`/`preset_install`, `check_updates` |
| `db` | `set`, `move`, `create`, `export`, `import`, `snapshot`, `snapshots`, `restore`, `snapshot_delete` |
| `env` | `setup`, `check`, `override` |
| `runtime` | `versions`, `node_install`/`node_uninstall`, `php_list`, `ext_list`/`ext_add`/`ext_remove`, `ports_list`/`ports_add`/`ports_remove` |
| `worker` | `list`, `start`, `stop`, `add`, `remove`, `health`, `heal`, `mode_get`/`mode_set`, `queue_start`/`queue_stop`, `horizon_start`/`horizon_stop`, `reverb_start`/`reverb_stop`, `schedule_start`/`schedule_stop`, `stripe_start`/`stripe_stop`, `stripe_config` |
| `exec` | `artisan`, `console`, `composer`, `vendor_bins`, `vendor_run`, `commands_list`, `commands_run`, `command_add`, `command_remove` |
| `framework` | `list`, `add`, `remove`, `prune`, `search`, `update`, `project_new`, `setup` |
| `diag` | `status`, `doctor`, `site_doctor`, `which`, `check`, `dns_diagnose`, `bug_report`, `analyze_queries`, `route_timing`, `optimize_route`, `dumps_recent`/`dumps_status`/`dumps_clear`/`dumps_toggle`, `profiler_toggle`/`profiler_status`/`profiler_clear`/`profiler_report`, `xdebug_on`/`xdebug_off`/`xdebug_status` |
| `logs` | `sources` (list first), then `fetch` with `source` + `grep` + `since`/`until` + `level` + `lines`. Each `fetch` returns a `cursor`; pass it back as `since` to stream new lines only. |
| `worktree` | `list`, `add`, `remove`, `db_isolate`, `db_share` |

## logs tool — streaming via cursor

The `logs` tool is request/response, but supports streaming:
1. `action: "sources"` → list all sources (`app:<file>`, `fpm`, `worker:<name>`, `nginx`, `dns`, `watcher`, `ui`, `<service>`, `php<ver>`).
2. `action: "fetch"` with `source`, optional `grep` (regex or substring), `since`/`until` (relative like `15m`/`2h30m`, or a timestamp), `level` (app logs only), `lines`.
3. The response includes an opaque `cursor`. Next call, pass it as `since` to get only lines newer than the last fetch.

## Path/site resolution

The MCP server resolves which site an operation targets by:
1. Explicit `path` argument (if the tool accepts one)
2. `LERD_SITE_PATH` environment variable
3. The directory the AI client opened / current working directory

Usually you don't pass `path` — the current project dir is the context.

## Agent variable passthrough

When PHP runs in the container (via `lerd php` / `lerd artisan` / tinker / MCP `exec`), Lerd forwards host-side agent environment variables (`CLAUDECODE`, `AI_AGENT`, `CURSOR_AGENT`, `GEMINI_CLI`, …) into the container. This lets packages like `laravel/pao` return compact JSON when they detect an agent. For commands run via `exec` with no real agent variable, Lerd injects a neutral `AI_AGENT=lerd-mcp` marker.
