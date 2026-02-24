# 01 — System Overview

## 1.1 What IronClaw Is

IronClaw is a **secure, self-expanding personal AI assistant**. It receives user messages from multiple channels (terminal UI, web browser, HTTP webhooks, WASM-based chat platforms), routes them through an LLM reasoning loop, executes tools in sandboxed environments, and streams results back in real time. Its defining properties:

- **Single-owner model** — one authenticated user per deployment, not multi-tenant.
- **Defense in depth** — prompt injection sanitization, secret leak detection, tool approval gates, and zero-exposure credential injection.
- **Self-expanding** — the agent can build new WASM tools at runtime and install skills from a registry, extending its own capabilities without redeployment.

---

## 1.2 Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| **Language** | Rust (2021 edition) | Async throughout via `tokio` runtime |
| **Web framework** | Axum | HTTP API, SSE, WebSocket, middleware |
| **Terminal UI** | Ratatui | Full TUI with overlays, composer, approval dialogs |
| **Frontend (web)** | Vanilla HTML / CSS / JS | Single-page app served from `src/channels/web/static/` |
| **Primary DB** | PostgreSQL | `pgvector` for embeddings, `tsvector` for FTS, `refinery` migrations |
| **Embedded DB** | libSQL / Turso | SQLite dialect with FTS5 and `libsql_vector_idx`; zero-dependency local mode |
| **WASM runtime** | Wasmtime | Sandboxed tool execution with fuel metering and memory limits |
| **Container sandbox** | Docker | Isolated job execution with network proxy and credential injection |
| **Real-time** | SSE + WebSocket | `tokio::sync::broadcast` channel (buffer: 256) with atomic connection tracking |
| **Serialization** | Serde (JSON) | All API types, config, DB serialization |
| **Error handling** | `thiserror` | Typed error hierarchy, no `.unwrap()` in production |
| **Cryptography** | AES-256-GCM, BLAKE3 | Secrets encryption, WASM binary integrity verification |

### Compile-Time Feature Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `postgres` | Yes | PostgreSQL backend |
| `libsql` | No | libSQL/Turso backend |

Both can be enabled simultaneously; runtime selection via `DATABASE_BACKEND` env var.

---

## 1.3 Roles & Permissions

IronClaw is **not multi-tenant**. There are no user accounts or RBAC. Instead, there are three trust contexts that determine what capabilities are available:

### Permission Matrix

| Trust Context | Authentication | Tool Access | Filesystem | Network |
|---------------|---------------|-------------|------------|---------|
| **Owner** | Bearer token (constant-time comparison via `subtle` crate) | All tools; can approve/deny individual tool executions | Full (via sandbox policy) | Full (via sandbox policy) |
| **Trusted Skill** | Implicit — user-placed in `~/.ironclaw/skills/` or workspace `skills/` | All tools available to the agent | Inherits owner sandbox | Inherits owner sandbox |
| **Installed Skill** | Downloaded from ClawHub registry | **Read-only subset only**: `memory_search`, `memory_read`, `memory_tree`, `time`, `echo`, `json`, `skill_list`, `skill_search` | No write access | No network tools |

**Critical attenuation rule:** If *any* installed (registry-sourced) skill is active in the current LLM context, the tool ceiling drops to the read-only subset for *all* skills in that turn — even trusted ones. This is a defense-in-depth measure against prompt injection through skill content.

### Tool Approval Flow

The owner can control tool execution granularity:

1. **Per-execution approval** (default when `auto_approve_tools = false`) — each tool call triggers an `ApprovalNeeded` event broadcast to all connected clients.
2. **Always-approve** — the owner can respond with `"always"` to auto-approve future calls to that tool.
3. **Deny** — blocks the tool call; the agent receives a `ToolError::AuthRequired` and must adapt.

Approval channels:
- **TUI**: Overlay dialog with approve/always/deny options
- **Web**: `POST /api/chat/approval` with `{ request_id, action, thread_id }`
- **WebSocket**: JSON message with `type: "Approval"`

---

## 1.4 Infrastructure Architecture

### Job Scheduling

- **No external queue** — jobs are tracked in an in-process `Arc<RwLock<HashMap<Uuid, ScheduledJob>>>`.
- Each job gets a dedicated `tokio::spawn` worker task communicating over an `mpsc::channel(16)`.
- Max parallel jobs: **5** (configurable via `agent.max_parallel_jobs`).
- Capacity enforcement is TOCTOU-safe: write-locked during the entire check-insert sequence.
- Automatic cleanup task polls `handle.is_finished()` every 1 second.

### Caching

| Cache | Type | Capacity | TTL | Eviction |
|-------|------|----------|-----|----------|
| LLM Response Cache | In-memory `HashMap` + `Mutex` | 1,000 entries | 1 hour | LRU by `last_accessed` timestamp; expired entries pruned on insert |
| WASM Module Cache | `Arc<HashMap>` | Unbounded | None | Compile-time only; lives for process lifetime |
| Skill Registry Cache | In-memory | — | 5 minutes | Per catalog query |

**Cache key** (LLM): SHA-256 of `model + messages_json + max_tokens + temperature + stop_sequences`. Tool-calling requests are **never cached** (side-effect protection).

### File & Blob Storage

| Artifact | Storage Location | Integrity |
|----------|-----------------|-----------|
| WASM binaries | DB (`wasm_tools.wasm_binary` BYTEA column) | BLAKE3 hash verified on every load |
| Workspace files | DB (`memory_documents` + `memory_chunks` tables) | Path-based hierarchy, chunked for search |
| Session tokens | `~/.ironclaw/session.json` (permissions `0o600`) | Also persisted to DB settings table |
| Installed tools | `~/.ironclaw/tools/{name}.wasm` + `{name}.capabilities.json` | File-based discovery with dev-mode artifact preference |
| Skills | `~/.ironclaw/skills/` (trusted), `~/.ironclaw/installed_skills/` (registry) | Trust level assigned by source directory |

### Real-Time Streaming

```
                    ┌─────────────────────────────────┐
                    │   tokio::broadcast::channel(256) │
                    │         (SseEvent enum)          │
                    └──────┬────────────┬──────────────┘
                           │            │
                    ┌──────▼──────┐  ┌──▼───────────────────┐
                    │ SSE Clients │  │ WebSocket Clients     │
                    │ (EventSource│  │ (dual-task per conn)  │
                    │  + keepalive│  │  Sender: mux SSE +    │
                    │  30s)       │  │    mpsc(64) direct    │
                    └─────────────┘  │  Receiver: routes to  │
                                     │    agent loop         │
                                     └──────────────────────┘
```

- **Max SSE connections**: 100 (atomic counter with `fetch_update`)
- **Keepalive**: 30-second SSE heartbeat
- **16 event types**: Response, Thinking, ToolStarted, ToolCompleted, ToolResult, StreamChunk, Status, ApprovalNeeded, AuthRequired, AuthCompleted, Error, JobStarted, JobMessage, JobToolUse, JobToolResult, JobStatus, JobResult, Heartbeat

### Container Sandbox

| Policy | Filesystem | Network | Use Case |
|--------|-----------|---------|----------|
| **ReadOnly** | `/workspace` (read-only mount) | Proxied allowlist only | Analysis, code review |
| **WorkspaceWrite** | `/workspace` (read-write mount) | Proxied allowlist only | Code generation, file edits |
| **FullAccess** | Full host filesystem | Unrestricted | Trusted admin tasks |

**Default resource limits**: Memory 2,048 MB, CPU shares 1,024, timeout 120s, output 64 KB.

**Network proxy** (host-side, per-container):
- All container HTTP/HTTPS routes through the proxy.
- Domain allowlist enforced (default: package registries, docs, GitHub, LLM APIs).
- HTTPS via CONNECT tunnel — proxy validates domain before establishing tunnel.
- **Credentials injected at transit time** by `CredentialResolver` — containers never hold raw secrets.

### LLM Provider Resilience

```
Request
  │
  ▼
┌─────────┐    ┌──────────────┐    ┌──────────┐
│  Retry  │───►│Circuit Breaker│───►│ Provider │
│ (3 max) │    │  (per-provider)│    │          │
└─────────┘    └──────────────┘    └──────────┘
                     │
              ┌──────▼──────┐
              │  Failover   │
              │   Chain     │
              └─────────────┘
```

- **Retry**: Exponential backoff — `1s × 2^attempt` with ±25% jitter, 100ms floor. Honors `Retry-After` headers on 429s.
- **Circuit breaker**: Opens after 5 consecutive transient failures. Recovery timeout 30s. Half-open requires 2 successful probes.
- **Transient errors** (trip breaker): `RequestFailed`, `RateLimited`, `InvalidResponse`, `SessionExpired`, `Http`, `Io`.
- **Non-transient** (don't trip): `AuthFailed`, `ContextLengthExceeded`, `ModelNotAvailable`.

### Rate Limiting

| Endpoint | Limit | Window |
|----------|-------|--------|
| `POST /api/chat/send` | 30 requests | 60 seconds (sliding window) |

---

## 1.5 Authentication

### Gateway Auth (Web/API)

- Single bearer token set via `GATEWAY_AUTH_TOKEN` env var.
- Checked in middleware via **constant-time comparison** (`subtle::ConstantTimeEq`).
- Two delivery methods: `Authorization: Bearer {token}` header, or `?token={token}` query parameter (for SSE `EventSource` which cannot set headers).
- WebSocket connections validated via Origin header (localhost only) plus token.

### LLM Provider Auth

| Provider | Auth Method | Token Source |
|----------|------------|--------------|
| NEAR AI (session) | `Authorization: Bearer sess_xxx` | Browser OAuth (GitHub/Google) → callback on localhost |
| NEAR AI (API key) | `Authorization: Bearer {key}` | `NEARAI_API_KEY` env var, persisted to `~/.ironclaw/.env` |
| OpenAI | `Authorization: Bearer sk-xxx` | `OPENAI_API_KEY` env var |
| Anthropic | `x-api-key: {key}` | `ANTHROPIC_API_KEY` env var |
| Ollama | None | Local, no auth |
| Tinfoil | `Authorization: Bearer {key}` | `TINFOIL_API_KEY` env var |

**Session renewal** (NEAR AI): On 401, a `Mutex<()>` prevents thundering herd during concurrent renewal attempts. Validation via `GET /v1/users/me`.

---

## 1.6 Configuration Hierarchy

**Priority** (highest to lowest):

1. Environment variables
2. TOML config file
3. Database settings table (per-user key-value)
4. Compiled defaults

### Setup Wizard (7 steps)

First-run onboarding configures:

| Step | Configures |
|------|-----------|
| 1 | Database backend (PostgreSQL vs libSQL) |
| 2 | Security (master key source: keychain / env / none) |
| 3 | LLM provider selection |
| 4 | Model selection |
| 5 | Embeddings (provider + model) |
| 6 | Channels (HTTP webhooks, Telegram, WASM channels, tunneling) |
| 7 | Heartbeat (proactive periodic execution) |

---

## 1.7 Error Architecture

Single top-level `Error` enum with domain-specific variants:

```
Error
├── ConfigError
├── DatabaseError
├── ChannelError
├── LlmError
├── ToolError
├── SafetyError
├── JobError
├── EstimationError
├── EvaluationError
├── RepairError
├── WorkspaceError
├── OrchestratorError
├── WorkerError
└── RoutineError
```

All use `thiserror` derive macros. Production code enforces zero `.unwrap()` / `.expect()` — errors are propagated with `.map_err()` context.
