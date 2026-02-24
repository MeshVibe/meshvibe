# mesh-vibe

Composable AI-powered tools that work together. Build your own personal ops ecosystem — each tool is a standalone CLI that plugs into a shared infrastructure of secrets, scheduling, notifications, and discovery.

Built for tinkerers. Every piece works alone, but they're better together.

## Architecture

mesh-vibe is conventions, not a framework. Every tool follows the same shape:

```
~/IdeaProjects/ProjectName/        # source code
~/mesh-vibe/data/project-name/     # runtime data, reports, state
~/mesh-vibe/heartbeat/             # scheduled task config
```

Shared infrastructure:
- **vault** — secrets via macOS Keychain
- **heartbeat** — scheduled task runner
- **event-log** — shared event journal
- **notify** — unified alerting
- **registry** — service catalog and discovery
- **dashboard** — unified HTML dashboard (planned)

## Projects

### Infrastructure

| Project | Description | CLI | Status |
|---------|-------------|-----|--------|
| [vault](https://github.com/mesh-vibe/vault) | macOS Keychain secret manager | `vault` | Stable |
| [heartbeat](https://github.com/mesh-vibe/heartbeat) | Scheduled task runner for Claude Code | `heartbeat` | Stable |
| [event-log](https://github.com/mesh-vibe/event-log) | Shared event journal | `eventlog` | Stable |
| [registry](https://github.com/mesh-vibe/registry) | Service catalog and health checks | `registry` | Stable |
| [notify](https://github.com/mesh-vibe/notify) | Unified notification routing | `notify` | Stable |
| [dashboard](https://github.com/mesh-vibe/dashboard) | Unified HTML dashboard | `dashboard` | Planned |

### Bots

| Project | Description | CLI | Data Dir | Status |
|---------|-------------|-----|----------|--------|
| [news-bot](https://github.com/mesh-vibe/news-bot) | Personal news aggregation from RSS + browser history | `newsbot` | `~/mesh-vibe/data/news-bot/` | Stable |
| [security-bot](https://github.com/mesh-vibe/security-bot) | Autonomous security scanning and threat detection | `securitybot` | `~/mesh-vibe/data/security-bot/` | Planned |
| [cost-bot](https://github.com/mesh-vibe/cost-bot) | API spend monitoring and anomaly detection | `costbot` | `~/mesh-vibe/data/cost-bot/` | Planned |
| [sysadmin-bot](https://github.com/mesh-vibe/sysadmin-bot) | Autonomous service supervisor | `sysadminbot` | `~/mesh-vibe/data/sysadmin-bot/` | Planned |
| [standards-bot](https://github.com/mesh-vibe/standards-bot) | Convention enforcement against mesh-vibe standards | `standardsbot` | `~/mesh-vibe/data/standards-bot/` | Planned |

### Interfaces

| Project | Description | CLI | Status |
|---------|-------------|-----|--------|
| [uterm](https://github.com/mesh-vibe/uterm) | Terminal with Claude co-pilot | `uterm` | Stable |
| [voice-claude](https://github.com/mesh-vibe/voice-claude) | Voice interface for Claude | `voiceclaude` | Stable |
| [remote-companion](https://github.com/mesh-vibe/remote-companion) | Mobile app for the ecosystem | — | In Progress |

## Conventions

All projects follow these rules. AI agents building new tools should match them exactly.

### Naming
- Repo names use **kebab-case** (hyphenated lowercase): `news-bot`, `event-log`, `sysadmin-bot`
- CLI commands use **no hyphens** for brevity: `newsbot`, `eventlog`, `sysadminbot`
- Package names use the `meshvibe-` prefix: `meshvibe-newsbot`, `meshvibe-registry`

### Stack
- TypeScript strict, ESM (`"type": "module"`)
- `tsconfig.json`: target ES2022, module Node16, moduleResolution Node16
- `src/` source, `dist/` output, `tests/` for tests
- npm (not yarn or bun), vitest for testing, commander for CLIs
- Node.js >= 20

### Package structure
```
project-name/
├── manifest.md          # project manifest (see below)
├── package.json         # bin field for CLI, "type": "module"
├── tsconfig.json
├── tsconfig.build.json  # extends tsconfig, excludes tests
├── LICENSE              # MIT
├── README.md
├── src/
│   ├── index.ts         # library exports
│   ├── cli.ts           # CLI entry point (#!/usr/bin/env node)
│   └── templates/
│       └── skill.md.ts  # Claude Code skill installer
└── tests/
    └── *.test.ts
```

### Scripts
```json
{
  "build": "tsc -p tsconfig.build.json",
  "dev": "tsc -p tsconfig.build.json --watch",
  "test": "vitest run",
  "lint": "tsc --noEmit"
}
```

### Directory layout
```
~/mesh-vibe/
├── data/
│   ├── news-bot/         # runtime data per bot
│   ├── security-bot/
│   ├── cost-bot/
│   ├── sysadmin-bot/
│   ├── event-log/
│   └── dashboard/
└── heartbeat/            # scheduled task configs
    ├── config.md
    └── *.md              # task files
```

### Secrets
Use vault. Never hardcode keys. In heartbeat tasks use the `vault://` prefix:
```yaml
env:
  API_KEY: "vault://my-api-key"
```

### Scheduling
Add a heartbeat task file in `~/mesh-vibe/heartbeat/`:
```yaml
---
schedule: daily at 05:00
timeout: 30m
dir: ~
enabled: true
env:
  ANTHROPIC_API_KEY: "vault://anthropic-api-key"
---

Your prompt here.
```

### Events
Bots emit structured events via event-log:
```bash
eventlog emit "news-bot.scan_complete" --priority low --data '{"articles": 25}'
```

Standard events every bot should emit:
- `<bot>.started` — task began
- `<bot>.completed` — task finished successfully
- `<bot>.failed` — task failed
- `<bot>.error` — unexpected error

### Reports
Bots that produce HTML reports write them to their data dir:
- Path: `~/mesh-vibe/data/<bot>/report.html`
- Dashboard aggregates all reports into a single view
- Declare reports in the manifest so dashboard and registry can discover them

### Notifications
Bots declare `notify_on` events in their manifest. notify reads the registry to find available channels and routes alerts based on priority:
- **critical** — SMS + push + macOS notification
- **high** — push + macOS notification
- **medium** — macOS notification
- **low** — logged only, visible in dashboard

### Claude Code Skills
Each project installs a skill via `<cli> init` to `~/.claude/skills/<name>/SKILL.md`. Skills use YAML frontmatter with `name` and `description` fields.

## Manifest Format

Every project has a `manifest.md` in its root. This is how the registry discovers and understands services.

```yaml
---
name: project-name
description: One-line description
cli: command-name
data_dir: ~/mesh-vibe/data/project-name
version: 0.1.0
reports:
  - name: Report Name
    path: ~/mesh-vibe/data/project-name/report.html
health_check: command-name status
depends_on:
  - vault
notify_on:
  - event: task_complete
    priority: low
  - event: task_failed
    priority: high
---

Longer description of what this project does and how it works.
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Project identifier, kebab-case, matches repo name |
| `description` | yes | One-line summary |
| `cli` | yes | CLI command name (no hyphens, after npm link) |
| `data_dir` | no | Runtime data directory under `~/mesh-vibe/data/` |
| `version` | yes | Semver |
| `reports` | no | HTML reports this project generates |
| `health_check` | no | Command to verify the service is healthy |
| `depends_on` | no | Other mesh-vibe projects this depends on |
| `notify_on` | no | Events that should trigger notifications |

## Adding a New Bot

1. Create the project following the package structure above
2. Add a `manifest.md` to the project root
3. Build and link: `npm install && npm run build && npm link`
4. Install the skill: `<cli> init`
5. Add a heartbeat task if it runs on a schedule
6. Push to the mesh-vibe org

The registry will discover it automatically from the manifest.

## Quick Reference

```bash
# Secrets
vault set <key> <value>
vault get <key>
vault list

# Scheduling
heartbeat status
heartbeat list
heartbeat run <task>
heartbeat history

# Events
eventlog emit "<bot>.<event>" --priority <level>
eventlog query --source <bot> --since 24h
eventlog tail

# Notifications
notify send <message> --priority <level>
notify channels

# Dashboard
dashboard open

# News
newsbot scan
newsbot status

# Terminal
uterm

# Voice
voiceclaude
```

## License

MIT — all projects.
