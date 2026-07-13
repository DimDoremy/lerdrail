# Installing lerdrail

`lerdrail` is a 5-skill harness pack (plain Markdown). Two install paths — pick one.

## Option A — `npx skills` (recommended)

[`npx skills`](https://github.com/vercel-labs/skills) is the open agent-skills installer ("npm for AI agents"). It auto-detects your coding agent (ZCode, Claude Code, Codex, Cursor, …) and installs the skill directories into the right location.

### Global (use across all projects)

```bash
npx skills add DimDoremy/lerdrail --global
```

Installs all 5 skills into your user-level skills directory (e.g. `~/.zcode/skills/`), available in every project.

### Project-local (only this project)

```bash
# from your Laravel project root
npx skills add DimDoremy/lerdrail
```

Installs into the project's skills directory (e.g. `.zcode/skills/`) and writes a `skills-lock.json` so teammates get the same set. Commit that lockfile.

### A single skill only

```bash
npx skills add DimDoremy/lerdrail --skill postgres-valkey --global
```

Skills available: `lerd-dev`, `laravel-conventions`, `postgres-valkey`, `react-tailwind`, `lerdrail-workflow`. List them without installing:

```bash
npx skills add DimDoremy/lerdrail --list
```

### Managing installed skills

```bash
npx skills list                 # what's installed (add -g for global)
npx skills update               # pull latest
npx skills remove postgres-valkey -g   # remove one
```

> The default install symlinks into your agent's skills dir. Add `--copy` if you'd rather have file copies (e.g. to edit them).

## Option B — git clone (vendor into the repo)

If you prefer the pack to live **inside** a project and travel with its git history (clone-and-go for teammates, same philosophy as `lerd mcp:inject`):

```bash
mkdir -p .zcode/skills
cd .zcode/skills
git clone https://github.com/DimDoremy/lerdrail.git
# or, to vendor it flat into .zcode/skills/:
# cp -r lerdrail/* . && rm -rf lerdrail
```

Then commit the skill files. Nothing else to configure.

## Requirements (either path)

- A local **Lerd** install — https://lerd.sh
- A Laravel project with **Laravel Boost** (`composer require laravel/boost --dev && php artisan boost:install`)
- The **superpowers** plugin in your agent
- The **context7** MCP server configured
- An agent that loads Agent Skills (ZCode, Claude Code, …)

## Troubleshooting

| Symptom | Fix |
|---|---|
| `npx skills add` finds fewer than 5 skills | A skill's `description` is too long/complex and gets skipped during discovery — keep descriptions short (detail belongs in the SKILL.md body). This repo is already fixed; report if it recurs. |
| Skill installed but agent doesn't trigger it | The agent matches the `description` ("Use when…"); make sure your task phrasing aligns with it. |
| Want file copies instead of symlinks | Add `--copy` to the `add` command. |
