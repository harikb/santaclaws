# 01 — System Overview

## 1.1 Project Identity

**Name:** NanoClaw
**Type:** Personal AI assistant platform — a single-user orchestrator that bridges messaging channels (WhatsApp) to sandboxed Claude AI agents running in containers.

**Architecture Style:** Single-process event loop with polling-based message ingestion, file-based IPC, and OS-level container isolation. No HTTP server, no REST API, no frontend — the "UI" is WhatsApp itself.

---

## 1.2 Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| **Runtime** | Node.js 20+ / TypeScript 5.7 | ES2022 modules, strict mode, single process |
| **Database** | SQLite via `better-sqlite3` | Synchronous API, no ORM, migrations in code |
| **Messaging** | `@whiskeysockets/baileys` v7 | Unofficial WhatsApp Web client (QR auth) |
| **AI Engine** | Claude Agent SDK (Claude Code) | Runs inside containers with session persistence |
| **Containers** | Docker (Linux) / Apple Container (macOS) | Per-group sandboxing with volume mounts |
| **Scheduling** | `cron-parser` | Cron, interval, and one-shot task scheduling |
| **Logging** | `pino` + `pino-pretty` | Structured JSON logs, colored terminal output |
| **Validation** | `zod` | Runtime schema validation |
| **Process Mgmt** | `launchd` (macOS) / `systemd` (Linux) | Auto-restart, background daemon |

**No frontend, no HTTP server, no message broker, no cache layer.** The system is intentionally minimal.

---

## 1.3 Roles & Permissions

NanoClaw is a **single-user system** — there is no authentication, no login, no multi-tenancy. The "owner" is the person who runs the process. Authorization is structural: it is enforced by which filesystem directory a container runs in, not by tokens or credentials.

### Role Definitions

| Role | Identity Mechanism | Description |
|------|-------------------|-------------|
| **Owner** | Implicit (runs the process) | The single human user. All messages originate from or are addressed to them. |
| **Main Group** | Folder name = `"main"` | The owner's self-chat (or designated primary channel). Unrestricted privileges. |
| **Regular Group** | Any other registered folder | A WhatsApp group chat. Restricted to its own namespace. |

### Permissions Matrix

| Capability | Main Group | Regular Group |
|-----------|:----------:|:-------------:|
| Receive all messages without trigger | Yes | No — requires `@Andy` mention |
| Send messages to own chat | Yes | Yes |
| Send messages to any chat | Yes | No |
| Schedule tasks for own group | Yes | Yes |
| Schedule tasks for other groups | Yes | No |
| Pause/resume/cancel own tasks | Yes | Yes |
| Pause/resume/cancel any task | Yes | No |
| Register new groups | Yes | No |
| Refresh WhatsApp group metadata | Yes | No |
| Read project source code | Yes (read-only mount) | No |
| Read global shared memory | Yes | Yes (read-only) |
| Write to global shared memory | Yes | No |
| Mount additional host directories | Yes (validated) | Yes (forced read-only if `nonMainReadOnly`) |
| Modify security allowlist | No (external file) | No (external file) |

### How Authorization Works

Authorization is **not checked at the application layer** in the traditional sense. Instead:

1. **Container filesystem isolation** — each group's container only has mounts for its own directory. A regular group literally cannot see other groups' files.
2. **IPC namespace isolation** — each group writes to `/data/ipc/{groupFolder}/`. The IPC watcher determines the source group from the directory path, not from any claim in the message payload.
3. **Main group detection** — a simple string comparison: `sourceGroup === "main"`. The main group's IPC directory is `/data/ipc/main/`.

This is a **capability-based security model**: containers can only do what their filesystem mounts allow. There are no tokens to steal or bypass.

---

## 1.4 Infrastructure Architecture

### Process Model

```
┌─────────────────────────────────────────────────┐
│              NanoClaw (single Node.js process)   │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Message   │  │ Task     │  │ IPC Watcher   │  │
│  │ Loop      │  │ Scheduler│  │ (1s poll)     │  │
│  │ (2s poll) │  │ (60s)    │  │               │  │
│  └─────┬─────┘  └────┬─────┘  └───────┬───────┘  │
│        │              │                │          │
│        └──────┬───────┘                │          │
│               ▼                        │          │
│  ┌────────────────────┐                │          │
│  │    Group Queue     │◄───────────────┘          │
│  │  (max 5 concurrent)│                           │
│  └────────┬───────────┘                           │
│           │                                       │
│    ┌──────┼──────┐                                │
│    ▼      ▼      ▼                                │
│  ┌───┐  ┌───┐  ┌───┐   ... up to 5               │
│  │ C │  │ C │  │ C │   containers                 │
│  └───┘  └───┘  └───┘                              │
└─────────────────────────────────────────────────┘
```

### Polling Loops (No Event Bus)

The system uses three independent polling loops, not event-driven pub/sub:

| Loop | Interval | Purpose |
|------|----------|---------|
| **Message Loop** | 2,000 ms | Query SQLite for new messages across all registered groups |
| **Task Scheduler** | 60,000 ms | Check `scheduled_tasks` for due items (`next_run <= now`) |
| **IPC Watcher** | 1,000 ms | Scan `/data/ipc/*/messages/` and `/data/ipc/*/tasks/` for new JSON files |

### Concurrency & Queuing

- **Global limit:** max 5 containers running simultaneously (`MAX_CONCURRENT_CONTAINERS`)
- **Per-group queue:** each group has a queue of pending messages and tasks
- **Priority:** scheduled tasks drain before pending messages
- **Backoff:** failed processing retries with exponential delay: 5s → 10s → 20s → 40s → 80s (max 5 retries)
- **Idle containers:** after completing work, containers stay alive for 30 minutes waiting for follow-up messages. If new messages arrive, they're piped to the running container via IPC input files.

### File-Based IPC

There is no message broker (no Redis, no RabbitMQ, no Unix sockets). Communication between the orchestrator and containers uses **atomic file writes**:

```
data/ipc/
├── main/                    # Main group namespace
│   ├── messages/            # Outbound messages (container → orchestrator → WhatsApp)
│   ├── tasks/               # Task management commands (schedule, pause, cancel)
│   └── input/               # Inbound follow-up messages (orchestrator → container)
├── {groupFolder}/           # Per-group namespace (same structure)
└── errors/                  # Failed IPC files moved here for debugging
```

**Atomic write protocol:** write to `{name}.tmp` → rename to `{name}.json`. This prevents the watcher from reading partial files.

### Storage Layout

```
nanoclaw/
├── store/
│   ├── auth/                # WhatsApp auth state (multi-file)
│   └── db.sqlite            # All application state
├── data/
│   ├── ipc/                 # File-based IPC (see above)
│   └── sessions/            # Per-group Claude session directories
│       └── {group}/.claude/ # Session files + settings.json
├── groups/
│   ├── main/                # Main group's workspace
│   │   ├── CLAUDE.md        # Agent memory/instructions
│   │   └── logs/            # Container execution logs
│   ├── {group}/             # Per-group workspace
│   │   ├── CLAUDE.md        # Group-specific memory
│   │   └── logs/
│   └── global/              # Shared read-only memory (reserved name)
└── .env                     # Secrets (never mounted into containers)
```

### Secrets Management

Secrets follow a strict containment protocol:

1. **Stored** in `.env` at project root (never committed)
2. **Parsed** by `env.ts` — reads file directly, does NOT load into `process.env`
3. **Injected** via container stdin as JSON (`{ secrets: { ANTHROPIC_API_KEY: "..." } }`)
4. **Deleted** from the input object immediately after writing to stdin
5. **Never** written to disk, mounted as files, set as env vars, or logged

Only two secrets are currently used: `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY`.

---

## 1.5 Startup & Shutdown

### Startup Sequence

| Step | Action | Failure Mode |
|------|--------|-------------|
| 1 | Verify container runtime (`docker info`) | Fatal exit with help message |
| 2 | Kill orphan containers from previous runs | Warning, continue |
| 3 | Initialize SQLite (run migrations) | Fatal exit |
| 4 | Load state: cursors, sessions, registered groups | Fatal exit |
| 5 | Register SIGTERM/SIGINT shutdown handlers | — |
| 6 | Connect WhatsApp channel (QR auth if needed) | Fatal exit |
| 7 | Start scheduler loop (60s interval) | Fatal exit |
| 8 | Start IPC watcher (1s interval) | Fatal exit |
| 9 | Recover unprocessed messages from before crash | Warning if none |
| 10 | Start message loop (2s interval) | Fatal exit |

### Graceful Shutdown

1. Signal received (SIGTERM/SIGINT)
2. `queue.shutdown(10000)` — 10-second grace period for active containers
3. Active containers are **not killed** — they finish naturally (detached with `--rm`)
4. Disconnect WhatsApp channel
5. `process.exit(0)`

### Crash Recovery

On startup, the system scans all registered groups for messages received after each group's `last_agent_timestamp`. If unprocessed messages exist, they are re-enqueued. This handles the case where the process crashed between advancing the message cursor and completing agent processing.

---

## 1.6 Security Boundaries

| Boundary | Mechanism |
|----------|-----------|
| **Container ↔ Host** | OS-level container isolation (Docker/Apple Container) |
| **Group ↔ Group** | Separate filesystem mounts, separate IPC namespaces |
| **Agent ↔ Secrets** | stdin-only injection, never on disk |
| **Agent ↔ Source Code** | Main group: read-only mount. Others: no access. |
| **Agent ↔ Credentials** | Mount allowlist blocks `.ssh`, `.aws`, `.gnupg`, etc. |
| **Agent ↔ Allowlist** | Allowlist stored at `~/.config/nanoclaw/`, outside project root, never mounted |
| **Output ↔ Memory** | 10 MB stdout/stderr cap prevents memory exhaustion |
