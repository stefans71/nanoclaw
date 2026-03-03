# NanoClaw — Spec

_Last updated: 2026-03-01_

## What We're Building

NanoClaw is a personal AI assistant running as a single Node.js process. It connects to WhatsApp (via Baileys), routes messages to Claude Agent SDK running inside isolated Linux containers (Docker), and sends responses back. Each WhatsApp group gets its own container, filesystem, and memory — isolation is at the OS level, not application-level permission checks.

The codebase is intentionally small: ~15 source files, no microservices, no configuration sprawl. Customization happens by modifying code or running Claude Code skills (`/setup`, `/customize`, `/debug`), not by filling config forms.

## Users

One user (the owner). WhatsApp is the UI. The owner talks to the assistant from their phone; the assistant runs on a server or always-on machine.

## Core Flows

1. **Message handling:** WhatsApp message → SQLite → poll loop (2s) → trigger check (`@Andy`) → format context → spawn container → Claude Agent SDK → response → WhatsApp
2. **Scheduled tasks:** cron/interval/once → task scheduler (60s) → spawn container → agent runs → optional `send_message`
3. **IPC:** Container writes to `data/ipc/` → IPC watcher reads and routes group registrations and messages back to host
4. **Memory:** Agent cwd is `groups/{group}/`, CLAUDE.md files load automatically via `settingSources: ['project']`, agent writes to `./CLAUDE.md` to persist

## Architecture

Single process. Three concurrent loops: message poller, task scheduler, IPC watcher. Agents run in Docker containers spawned on demand, kept alive for 30min idle timeout. Per-group queue (`GroupQueue`) prevents concurrent processing of the same group.

```
WhatsApp (baileys) → SQLite → poll loop → GroupQueue → Docker container (Claude Agent SDK) → response
                                        ↑
                         task scheduler ┘
```

Full technical reference: `docs/SPEC.md`. Known issues and debug commands: `docs/DEBUG_CHECKLIST.md`.

## Data Model

| Table | Purpose |
|-------|---------|
| `messages` | All WhatsApp messages; queried by chat_jid and timestamp |
| `registered_groups` | Active groups with folder, name, container_config |
| `sessions` | Per-group Claude session IDs for conversation continuity |
| `scheduled_tasks` | cron/interval/once tasks with status and next_run |
| `router_state` | Key/value: last_timestamp, last_agent_timestamp (per-group JSON) |
| `task_run_logs` | History of scheduled task executions |

All stored in `store/messages.db`.

## Integrations

- **WhatsApp** via @whiskeysockets/baileys (WA Web protocol)
- **Claude Agent SDK** run via `claude` CLI inside Docker containers
- **Docker** for container runtime (Linux host; Apple Container optional on macOS)
- **SQLite** via better-sqlite3

## Scope

### In scope
- WhatsApp as primary channel
- Docker container runtime (Linux host)
- Scheduled tasks (cron, interval, one-time)
- Per-group memory and filesystem isolation
- Claude Code as the development interface

### Explicitly out of scope
- Multi-user support
- Web UI or monitoring dashboard
- Configuration files (customization = code changes)
- New features merged to base (add skills instead: `/add-telegram`, etc.)

## Open Questions

- [MED] IDLE_TIMEOUT == CONTAINER_TIMEOUT (both 30min) — containers always exit via SIGKILL instead of graceful shutdown
- [MED] Message cursor advanced before agent completes — container timeout causes permanent message loss
- [LOW] systemd service configuration for this Linux install not yet verified

## Decisions Log

- **Docker over Apple Container** — running on Linux; Docker is the cross-platform default
- **SQLite not file-based state** — atomic, queryable, crash-safe
- **No configuration files** — customization = code changes; keeps codebase honest and auditable
- **Skills not PRs** — new capabilities contributed as SKILL.md files, not merged features; each install stays clean
