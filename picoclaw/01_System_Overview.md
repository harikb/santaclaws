# 01 — System Overview

> **Project:** PicoClaw
> **Type:** Ultra-lightweight, multi-channel AI agent framework
> **Tagline:** Personal AI assistant in Go — <10 MB RAM, <1 s startup, runs on $10 hardware

---

## 1. Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| **Language** | Go 1.25 | Single static binary, no runtime dependencies |
| **HTTP Server** | `net/http` (stdlib) | Gateway serves webhooks and health endpoints |
| **CLI Framework** | Custom command dispatcher | Subcommands: `agent`, `gateway`, `cron`, `auth`, `onboard`, `migrate`, `skills`, `status` |
| **LLM SDKs** | `anthropic-sdk-go`, `openai-go/v3` | OpenAI SDK doubles as compatibility layer for 15+ providers |
| **Message Channels** | 12+ platform SDKs | telego, discordgo, slack-go, oapi-sdk-go (Feishu), botgo (QQ), dingtalk-stream-sdk-go, etc. |
| **Config Parsing** | `caarlos0/env/v11` | Environment variable overrides on top of JSON config |
| **Scheduling** | `adhocore/gronx` | Cron expression evaluation |
| **Auth** | `golang.org/x/oauth2` + custom PKCE | OAuth flows for Google Cloud Code Assist, GitHub Copilot |
| **Build** | Make + multi-stage Docker (Alpine 3.23) | Targets: linux/{amd64,arm64,riscv64,loong64}, darwin/arm64, windows/amd64 |
| **Storage** | Filesystem only | JSON for sessions/state, Markdown for memory — no database engine |

---

## 2. Runtime Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Gateway Process                          │
│                                                                  │
│  ┌────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │  Channel    │    │  Message Bus │    │  Agent Registry      │  │
│  │  Manager    │───▶│  (in-memory) │───▶│                      │  │
│  │            │    │              │    │  ┌────────────────┐  │  │
│  │ Telegram   │    │ inbound[100] │    │  │ Agent Instance │  │  │
│  │ Discord    │    │ outbound[100]│    │  │  - Session Mgr │  │  │
│  │ Slack      │    │              │    │  │  - Tool Registry│  │  │
│  │ WeCom      │◀───│              │◀───│  │  - Skill Loader│  │  │
│  │ Feishu     │    └──────────────┘    │  │  - Provider    │  │  │
│  │ DingTalk   │                        │  └────────────────┘  │  │
│  │ QQ         │    ┌──────────────┐    │  ┌────────────────┐  │  │
│  │ LINE       │    │ Route        │    │  │ Agent Instance │  │  │
│  │ OneBot     │    │ Resolver     │───▶│  │  (subagent)    │  │  │
│  │ WhatsApp   │    │ (7-level)    │    │  └────────────────┘  │  │
│  │ MaixCAM    │    └──────────────┘    └──────────────────────┘  │
│  └────────────┘                                                  │
│                        ┌──────────────┐                          │
│                        │ Health Server│  GET /health, /ready     │
│                        │ :18790       │                          │
│                        └──────────────┘                          │
└──────────────────────────────────────────────────────────────────┘
```

### Message Flow

1. External platform delivers a webhook/event to a **Channel** adapter.
2. Channel publishes an `InboundMessage` to the **Message Bus** (buffered channel, cap 100).
3. The bus consumer passes the message to the **Route Resolver**.
4. The resolver runs the 7-level binding cascade and returns a `ResolvedRoute` (agent ID + session key).
5. The matched **Agent Instance** builds a system prompt, calls the LLM, and enters the tool-execution loop.
6. The agent publishes an `OutboundMessage` to the bus.
7. The Channel Manager's outbound dispatcher delivers the response to the originating channel.

---

## 3. User Roles & Permissions Matrix

PicoClaw does not have traditional user accounts. Identity is defined by the **channel + peer** tuple. Access is controlled through three independent mechanisms:

### 3.1 Channel-Level Allow Lists

Each channel config has an `allow_from` field — a list of platform-specific user IDs. If the list is non-empty, only those IDs may interact. An empty list means open access.

```json
"telegram": { "enabled": true, "allow_from": ["123456789"] }
```

| Behavior | `allow_from` empty | `allow_from` populated |
|---|---|---|
| Known user ID | Allowed | Allowed only if in list |
| Unknown user ID | Allowed | **Rejected** |

### 3.2 Binding-Based Agent Routing (7-Level Cascade)

Bindings determine **which agent** handles a message. They are evaluated top-to-bottom; first match wins.

| Priority | Match Level | Scope | Example Use Case |
|---|---|---|---|
| 1 | **Peer** | Exact user ID on a channel | "Route Alice on Telegram to the coding agent" |
| 2 | **Parent Peer** | Group/parent containing the user | "Route all members of group X to the support agent" |
| 3 | **Guild** | Server/workspace ID | "Route the entire Discord server to agent Y" |
| 4 | **Team** | Team ID | "Route team Z to a specialized agent" |
| 5 | **Account** | Non-wildcard account on channel | "Route a specific Slack workspace" |
| 6 | **Channel Wildcard** | Any user on channel (`*` account) | "Route all Telegram users to agent A" |
| 7 | **Default** | Global fallback | "Everything else goes to the main agent" |

**Implicit rule:** If no agents are configured at all, the system creates an implicit `"main"` agent as the default.

### 3.3 Subagent Spawn Permissions

A parent agent can spawn child agents during tool execution. This is gated by the parent's `subagents.allow_agents` list.

| Config Value | Effect |
|---|---|
| `["*"]` | Parent can spawn any registered agent |
| `["agent-a", "agent-b"]` | Parent can only spawn the named agents |
| `[]` or omitted | Parent cannot spawn subagents |

### 3.4 Tool-Level Restrictions

| Restriction | Config Key | Default | Effect |
|---|---|---|---|
| Workspace confinement | `restrict_to_workspace` | `true` | File operations (read, write, list, edit) are sandboxed to the agent's workspace directory |
| Shell deny patterns | `tools.exec.enable_deny_patterns` | `false` | When enabled, shell commands are matched against deny patterns before execution |
| Custom deny patterns | `tools.exec.custom_deny_patterns` | `[]` | Regex patterns that block matching shell commands |

### 3.5 Consolidated Permissions Matrix

| Actor | Can Send Messages | Agent Selection | File Access | Shell Access | Spawn Subagents |
|---|---|---|---|---|---|
| **Peer (allow-listed)** | Yes | Resolved via binding cascade | Agent's workspace only (if restricted) | Subject to deny patterns | Per parent agent config |
| **Peer (not in allow list)** | **No** (rejected at channel) | N/A | N/A | N/A | N/A |
| **Group member** | Yes (if channel allows) | Parent-peer or guild binding | Agent's workspace | Subject to deny patterns | Per parent config |
| **Wildcard channel user** | Yes (if no allow list) | Channel-level or default binding | Agent's workspace | Subject to deny patterns | Per parent config |
| **Subagent** | Via parent's channel context | Fixed at spawn time | Own workspace (may differ) | Inherits parent restrictions | Not recursive (no sub-subagents defined) |

---

## 4. Infrastructure & Storage

### 4.1 Message Bus

- **Type:** In-process, goroutine-safe buffered channels (`chan`, not an external broker).
- **Inbound buffer:** 100 messages.
- **Outbound buffer:** 100 messages.
- **Concurrency model:** Single dispatcher goroutine per direction; `sync.RWMutex` guards handler registration.
- **Failure mode:** If a channel send fails, the error is logged but the bus continues — no retry or dead-letter queue.

### 4.2 File-Based Storage

All persistence is filesystem-based. There is no database, no SQLite, no embedded KV store.

```
~/.picoclaw/
├── config.json                          # Main configuration
├── workspace/                           # Default agent workspace
│   ├── AGENT.md                         # Agent behavior guidelines
│   ├── IDENTITY.md                      # Agent identity definition
│   ├── USER.md                          # User context
│   ├── SOUL.md                          # Agent personality
│   ├── sessions/
│   │   ├── {sanitized_session_key}.json # Conversation history (atomic writes)
│   │   └── {sanitized_session_key}_summary
│   ├── state/
│   │   └── state.json                   # Last channel/chatID (atomic writes)
│   ├── memory/
│   │   ├── MEMORY.md                    # Long-term memory (persistent)
│   │   └── YYYYMM/
│   │       └── YYYYMMDD.md              # Daily notes
│   └── skills/
│       ├── weather/SKILL.md             # Installed skills
│       ├── github/SKILL.md
│       └── ...
└── skills/                              # Global skills (shared across agents)
```

**Write safety:** All state and session writes use the temp-file-then-rename pattern for crash safety on POSIX systems.

**Session key sanitization:** Colons in session keys are replaced with underscores for filesystem compatibility.

### 4.3 Configuration Hierarchy

Configuration is resolved in this order (later overrides earlier):

1. **Compiled defaults** (`pkg/config/defaults.go`) — base values for every field.
2. **Config file** (`~/.picoclaw/config.json`) — user-authored JSON.
3. **Environment variables** — pattern: `PICOCLAW_AGENTS_DEFAULTS_MODEL=gpt-4`.

### 4.4 Key Default Values

| Setting | Default | Notes |
|---|---|---|
| `agents.defaults.model` | `glm-4.7` (Zhipu) | Chinese LLM as default; most users will override |
| `agents.defaults.max_tokens` | `8192` | Output token limit per LLM call |
| `agents.defaults.temperature` | `nil` (provider default) | Not set unless explicitly configured |
| `agents.defaults.max_tool_iterations` | `20` | Hard cap on tool calls per turn |
| `agents.defaults.restrict_to_workspace` | `true` | Sandbox file operations |
| `session.dm_scope` | `"main"` | DMs share one session per agent (vs. `"peer"` for per-user) |
| `gateway.host` | `0.0.0.0` | Bind to all interfaces |
| `gateway.port` | `18790` | Gateway HTTP port |
| `heartbeat.enabled` | `true` | Periodic health pings |
| `heartbeat.interval` | `30` (minutes) | Heartbeat frequency |
| `tools.cron.exec_timeout_minutes` | `5` | Cron task execution ceiling |

### 4.5 Health & Readiness

| Endpoint | Method | Status Codes | Purpose |
|---|---|---|---|
| `/health` | GET | `200` always | Liveness probe (Docker HEALTHCHECK) |
| `/ready` | GET | `200` or `503` | Readiness probe — returns 503 if `ready=false` or any registered check fails |

The readiness endpoint supports pluggable checks via `RegisterCheck(name, fn)`. Each check contributes a pass/fail to the aggregate readiness state.

**Timeouts:** 5-second read/write timeout on the health HTTP server.

### 4.6 Caching

| Cache | Location | Strategy |
|---|---|---|
| Skill search results | In-memory (`search_cache.go`) | LRU, max 100 entries, TTL 3600 s |
| Sessions | In-memory map + disk | Loaded from disk at startup, written on every update |
| Provider fallback cooldowns | In-memory | Per-model failure tracking with exponential backoff |
| OAuth tokens | Disk (`auth/store.go`) | Stored and refreshed per provider |

There is no shared distributed cache. All caching is process-local.

---

## 5. Deployment Modes

| Mode | Command | Description |
|---|---|---|
| **Single-shot agent** | `picoclaw agent -m "message"` | Process one message and exit. Suitable for cron jobs, scripts, and CI. |
| **Long-running gateway** | `picoclaw gateway` | Start all enabled channels, consume the message bus, serve health endpoints. The primary production mode. |
| **Cron scheduler** | `picoclaw cron list/add/remove` | Manage scheduled tasks that invoke the agent at intervals. |
| **Interactive onboarding** | `picoclaw onboard` | Setup wizard for first-time configuration. |
| **Docker** | `docker compose up picoclaw-gateway` | Containerized gateway with volume-mounted workspace and config. |

---

## 6. Cross-Cutting Concerns

### Concurrency
Every shared data structure (sessions, state, tool registry, channel manager, agent registry, message bus) is guarded by `sync.RWMutex`. The system is designed for concurrent multi-channel operation within a single process.

### Error Classification (Provider Fallback)
The provider subsystem classifies LLM errors into categories that drive fallback behavior:

| Error Class | Retriable | Action |
|---|---|---|
| `Auth` | No | Fail immediately — credentials are wrong |
| `RateLimit` | Yes | Try next fallback model |
| `Billing` | No | Fail — account issue |
| `Timeout` | Yes | Try next fallback |
| `Overload` | Yes | Try next fallback |
| `Format` | No | Fail — request is malformed |

### Logging
Structured logging throughout. Tool executions log name, arguments, duration, and result size. Channel events log connection status and errors.
