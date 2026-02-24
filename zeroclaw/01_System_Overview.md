# 01 — System Overview

> ZeroClaw Technical Blueprint | File 1 of 5

---

## 1. Project Identity

| Field | Value |
|---|---|
| **Name** | ZeroClaw |
| **Type** | Autonomous AI agent runtime |
| **Primary language** | Rust 2021 (MSRV 1.87) |
| **Version** | 0.1.6 |
| **License** | Repository-governed |
| **Default port** | `42617` |
| **Config directory** | `~/.zeroclaw/` (overridable: `ZEROCLAW_CONFIG_DIR`, `ZEROCLAW_WORKSPACE`) |

---

## 2. Tech Stack

### 2.1 Backend

| Layer | Technology | Notes |
|---|---|---|
| Language | Rust 2021, edition 2021 | MSRV 1.87 |
| Async runtime | Tokio 1.42 | multi-thread, macros, net, process, fs, signal |
| HTTP server | Axum 0.8 | HTTP/1.1, JSON, WebSocket, query, macros |
| HTTP client | Reqwest 0.12 | rustls-tls, JSON, multipart, streaming, SOCKS proxy |
| Serialization | serde + serde_json | All config, API, and storage |
| Crypto | ChaCha20-Poly1305 (secrets), HMAC-SHA256 (webhooks), Ring (JWT) | |
| TLS | rustls 0.23 + webpki-roots | No OpenSSL dependency |
| WebSocket | tokio-tungstenite 0.28 | Discord, Lark, gateway WS |
| Email | lettre 0.11 (SMTP) + async-imap + mail-parser | |

### 2.2 Frontend

| Layer | Technology | Notes |
|---|---|---|
| Framework | React 19 | SPA, embedded into binary |
| Language | TypeScript 5.7 | |
| Build tool | Vite 6 | |
| CSS | TailwindCSS 4 | |
| Routing | React Router DOM 7 | |
| Icons | Lucide React | |
| Embedding | rust-embed 8 + mime_guess | Served from `/_app/*`, SPA fallback at `/*` |

**Pages:** Dashboard, AgentChat, Config, Cost, Cron, Doctor, Integrations, Logs, Memory, Tools.

### 2.3 Database

| Backend | Engine | Use Case |
|---|---|---|
| **SQLite** (default) | rusqlite 0.37, bundled | Memory store (`brain.db`), cron store, embedding cache |
| **PostgreSQL** (optional) | Feature-gated `memory-postgres` | Memory store (alternative) |
| **Markdown** | Flat files in workspace | Memory store (alternative) |
| **Lucid** | SQLite + markdown overlay | Write-through cache memory store |

SQLite configuration: WAL journal, NORMAL sync, 8 MB mmap, ~2 MB page cache, temp in memory. FTS5 full-text search with BM25 scoring. Vector embedding storage as BLOB.

### 2.4 Infrastructure

| Concern | Technology |
|---|---|
| Container | Docker multi-stage (builder → dev/release); distroless production image |
| Registry | `ghcr.io/zeroclaw-labs/zeroclaw` |
| CI/CD | GitHub Actions (build, test, fuzz, security audit, CodeQL, release, Docker publish, Homebrew) |
| Service management | systemd (Linux user service), OpenRC (Linux init), launchd (macOS), Task Scheduler (Windows) |
| Tunnels | Cloudflare Tunnel, ngrok, Tailscale Funnel, custom command |
| Observability | Prometheus (metrics at `/metrics`), OpenTelemetry OTLP (feature-gated), structured tracing logs |
| Dependency audit | cargo-audit 0.22.1, cargo-deny 0.18.5 |

### 2.5 Feature Flags (Cargo)

All features are opt-in (no defaults enabled):

| Feature | Enables |
|---|---|
| `hardware` | USB enumeration (`nusb`) + serial port (`tokio-serial`) |
| `channel-matrix` | Matrix/Element E2EE channel (`matrix-sdk`) |
| `channel-lark` | Lark/Feishu WebSocket channel (`prost` protobuf) |
| `memory-postgres` | PostgreSQL memory backend |
| `observability-otel` | OpenTelemetry OTLP export |
| `peripheral-rpi` | Raspberry Pi GPIO (`rppal`) |
| `browser-native` | Rust-native browser automation (`fantoccini`) |
| `sandbox-landlock` | Linux Landlock sandbox |
| `sandbox-bubblewrap` | Bubblewrap sandbox |
| `probe` | STM32/Nucleo debug probe (`probe-rs`) |
| `rag-pdf` | PDF ingestion for RAG (`pdf-extract`) |
| `whatsapp-web` | WhatsApp Web protocol (`wa-rs`) |

### 2.6 Release Profile

```
opt-level = "z"      # Optimize for binary size
lto = "fat"          # Full link-time optimization
codegen-units = 1    # Single codegen unit (max optimization)
strip = true         # Strip debug symbols
panic = "abort"      # No unwinding overhead
```

---

## 3. User Roles & Permissions Matrix

### 3.1 Access Roles

| Role | Authentication | Scope |
|---|---|---|
| **CLI Operator** | Local process (no auth) | Direct terminal access; full CLI command set |
| **Paired Device** | Bearer token (via one-time pairing code exchange at `/pair`) | Remote API access; all `/api/*` endpoints |
| **Channel User** | Per-channel auth (bot token, webhook secret, HMAC) | Messaging via Telegram/Discord/Slack/etc.; per-sender message history (50 msgs) |
| **Delegate Agent** | Internal (spawned by orchestrator) | Sub-agent with scoped `allowed_tools`, `max_depth` (default 3), `max_iterations` (default 10) |

### 3.2 Autonomy Levels

These apply to what the agent is allowed to do on behalf of any role:

| Level | Serde key | Shell | File Write | Tool Execution | Approval |
|---|---|---|---|---|---|
| **ReadOnly** | `"readonly"` | Denied | Denied | Read-only tools only | N/A |
| **Supervised** (default) | `"supervised"` | Allowed (within allowlist) | Allowed (within workspace) | All registered tools | Required for medium-risk ops |
| **Full** | `"full"` | Allowed (within allowlist) | Allowed (within workspace) | All registered tools | Never required |

### 3.3 Permissions Matrix

| Capability | ReadOnly | Supervised | Full |
|---|---|---|---|
| Read files in workspace | Yes | Yes | Yes |
| Read memory | Yes | Yes | Yes |
| Write files in workspace | No | Yes (auto or approval) | Yes |
| Execute shell commands | No | Yes (allowlist + approval for medium-risk) | Yes (allowlist enforced) |
| Execute high-risk commands (`rm`, `sudo`, `curl`, etc.) | No | Blocked by default | Blocked by default (`block_high_risk_commands`) |
| Access paths outside workspace | No | No (unless `allowed_roots` configured) | No (unless `allowed_roots` configured) |
| Access forbidden system paths | No | No (hardcoded deny) | No (hardcoded deny) |
| Store/forget memory | No | Yes | Yes |
| Browser automation | No | Yes (if enabled) | Yes (if enabled) |
| HTTP requests | No | Yes (if enabled) | Yes (if enabled) |
| Cron management | No | Yes | Yes |
| Delegate to sub-agent | No | Yes (if agents configured) | Yes (if agents configured) |

### 3.4 Auto-Approve & Always-Ask Lists

| List | Default Contents | Effect |
|---|---|---|
| `auto_approve` | `["file_read", "memory_recall"]` | These tools execute without user approval in Supervised mode |
| `always_ask` | `[]` (empty) | These tools always require approval, even in Full mode |
| `non_cli_excluded_tools` | `[]` (empty) | Tools hidden from non-CLI channels |

### 3.5 Shell Command Classification

| Risk Level | Commands | Behavior (Supervised) |
|---|---|---|
| **Low** | `ls`, `cat`, `grep`, `find`, `echo`, `pwd`, `wc`, `head`, `tail`, `date` | Allowed |
| **Medium** | `git` (commit/push/reset/merge/rebase), `npm/pnpm/yarn` (install/publish), `cargo` (add/publish), `touch`, `mkdir`, `mv`, `cp`, `ln` | Requires approval |
| **High** | `rm`, `mkfs`, `dd`, `shutdown`, `sudo`, `chmod`, `curl`, `wget`, `ssh`, `nc` + 20 more | Blocked by default |

**Shell safety rules (always enforced):**
- Subshells blocked: `` ` ``, `$(`, `${`, `<(`, `>(`
- Unquoted redirects blocked: `<`, `>`
- `tee` command blocked
- Bare `&` blocked
- Dangerous args blocked: `find -exec`, `git config`, `git alias`, `git -c`
- Commands split on `;`, `|`, `&&`, `||` — each segment checked independently

---

## 4. Security Architecture

### 4.1 Defense Layers

```
┌─────────────────────────────────────────────┐
│  Layer 1: Network Boundary                  │
│  - Bind to 127.0.0.1 by default            │
│  - Public bind requires explicit opt-in     │
│  - Tunnel required for external access      │
│  - Request body limit: 64 KB               │
│  - Request timeout: 30 seconds             │
├─────────────────────────────────────────────┤
│  Layer 2: Authentication                    │
│  - Device pairing (one-time code → bearer)  │
│  - Webhook HMAC (SHA-256)                   │
│  - Bearer token (constant-time comparison)  │
│  - Per-channel bot tokens                   │
├─────────────────────────────────────────────┤
│  Layer 3: Rate Limiting                     │
│  - Sliding window per IP                    │
│  - Pair endpoint: 10/min                    │
│  - Webhook endpoint: 60/min                 │
│  - Max tracking keys: 10,000 (LRU evict)   │
│  - Idempotency dedup: 300s TTL             │
├─────────────────────────────────────────────┤
│  Layer 4: Policy Enforcement                │
│  - Autonomy level gates                     │
│  - Command allowlist + risk classification  │
│  - Path validation (traversal, null bytes)  │
│  - Workspace confinement                    │
│  - Forbidden path denylist (18 paths)       │
│  - Action budget: 20/hour (default)         │
│  - Cost budget: $5/day, $10/day, $100/month │
├─────────────────────────────────────────────┤
│  Layer 5: Execution Sandbox (optional)      │
│  - Landlock (Linux kernel)                  │
│  - Bubblewrap (Linux userspace)             │
│  - Firejail (Linux userspace)               │
│  - Docker (container isolation)             │
│  - Auto-detect or explicit backend          │
├─────────────────────────────────────────────┤
│  Layer 6: Runtime Guards                    │
│  - OTP gating for sensitive tools           │
│  - Emergency stop (kill-all, network-kill,  │
│    domain-block, tool-freeze)               │
│  - Prompt injection defense (PromptGuard)   │
│  - Credential leak detection (LeakDetector) │
│  - Audit log (signed events optional)       │
└─────────────────────────────────────────────┘
```

### 4.2 Secrets Management

- Engine: ChaCha20-Poly1305 authenticated encryption
- Storage: `~/.zeroclaw/secrets.enc` (or configured path)
- File permissions: `0o600` (Unix)
- Encryption enabled by default (`secrets.encrypt = true`)
- Sensitive config fields masked in API responses: `api_key`, `bot_token`, `access_token`, `secret`, `app_secret`, `signing_secret`

### 4.3 Forbidden System Paths (Hardcoded)

```
/etc  /root  /home  /usr  /bin  /sbin  /lib  /opt
/boot  /dev  /proc  /sys  /var  /tmp
~/.ssh  ~/.gnupg  ~/.aws  ~/.config
```

### 4.4 Emergency Stop (E-Stop)

| Level | Effect |
|---|---|
| `KillAll` | Stop all agent activity |
| `NetworkKill` | Block all outbound network |
| `DomainBlock(domains)` | Block specific domain patterns |
| `ToolFreeze(tools)` | Freeze specific tools |

Resume requires OTP by default (`require_otp_to_resume = true`). State persisted to `~/.zeroclaw/estop-state.json`.

---

## 5. Infrastructure Architecture

### 5.1 Deployment Topology

```
                    ┌─────────────────┐
                    │   Internet      │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────┴─────┐ ┌─────┴─────┐ ┌─────┴─────┐
        │ Cloudflare │ │   ngrok   │ │ Tailscale │
        │  Tunnel    │ │  Tunnel   │ │  Funnel   │
        └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
              └──────────────┼──────────────┘
                             │
                    ┌────────┴────────┐
                    │  Gateway (Axum) │
                    │  :42617         │
                    ├─────────────────┤
                    │  Rate Limiter   │
                    │  Auth (Pairing) │
                    │  Idempotency    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────┴─────┐ ┌─────┴─────┐ ┌─────┴─────┐
        │   Agent   │ │  Channels │ │   Cron    │
        │ Orchestr. │ │ (20+ xpts)│ │ Scheduler │
        └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
              │              │              │
        ┌─────┴──────────────┴──────────────┴─────┐
        │              Shared Services             │
        ├──────────────────────────────────────────┤
        │  Provider (LLM)  │  Memory (SQLite/PG)   │
        │  Tools (exec)    │  Security (policy)    │
        │  Observer (metrics) │  Health (registry) │
        └──────────────────────────────────────────┘
```

### 5.2 Daemon Architecture

The daemon is a multi-component supervisor that spawns and monitors:

| Component | Purpose | Restart Policy |
|---|---|---|
| `state_writer` | Writes `daemon_state.json` every 5 seconds | Exponential backoff |
| `gateway` | HTTP/WS API server | Exponential backoff |
| `channels` | Messaging transport listeners (if configured) | Exponential backoff |
| `heartbeat` | Periodic health tasks (min 5-min interval) | Exponential backoff |
| `scheduler` | Cron job executor (if `cron.enabled`) | Exponential backoff |

**Backoff:** starts at `channel_initial_backoff_secs` (min 1s), doubles on failure up to `channel_max_backoff_secs`, resets on clean exit. Shutdown via `SIGINT` (Ctrl+C) aborts all handles.

### 5.3 Caching

| Cache | Type | Location | Eviction |
|---|---|---|---|
| SQLite page cache | In-process | Memory (~2 MB) | LRU (SQLite internal) |
| SQLite mmap | OS page cache | Memory (8 MB) | OS-managed |
| Embedding cache | SQLite table | `brain.db` → `embedding_cache` | LRU by `accessed_at` (default 10,000 entries) |
| Response cache | In-memory | Agent runtime | TTL-based (default 60 min, max 5,000 entries, disabled by default) |
| Rate limiter state | In-memory | `HashMap<IP, Vec<Instant>>` | Sliding window (60s) + periodic sweep (5 min) + LRU at 10,000 keys |
| Idempotency store | In-memory | `HashMap<key, Instant>` | TTL (300s) + LRU at 10,000 keys |
| SSE broadcast | In-memory channel | Tokio broadcast | Ring buffer (256 events) |

### 5.4 File Storage

| Artifact | Default Path | Purpose |
|---|---|---|
| Config | `~/.zeroclaw/config.toml` | Runtime configuration |
| Secrets | `~/.zeroclaw/secrets.enc` | Encrypted API keys/tokens |
| Brain DB | `{workspace}/memory/brain.db` | Memory entries, embeddings, FTS index |
| Cron DB | `{workspace}/memory/brain.db` (shared) | Scheduled jobs and run history |
| Audit log | `{workspace}/audit.log` | Security audit trail (max 100 MB) |
| Daemon state | `{config_dir}/../daemon_state.json` | Component health JSON |
| E-stop state | `~/.zeroclaw/estop-state.json` | Emergency stop persistence |
| Runtime trace | `{workspace}/state/runtime-trace.jsonl` | JSONL event log (0o600) |
| Memory snapshot | `{workspace}/MEMORY_SNAPSHOT.md` | Cold-boot hydration source |
| Markdown memory | `{workspace}/memory/*.md` | Flat-file memory backend |

### 5.5 Observability

| Backend | Metrics | Traces | Endpoint |
|---|---|---|---|
| **Prometheus** | 11 metrics (`zeroclaw_*` prefix: 5 counters, 3 histograms, 3 gauges) | No | `/metrics` (text exposition) |
| **OpenTelemetry** | 10 metrics + 4 span types | Yes (OTLP spans) | Configurable (default `localhost:4318`) |
| **Log** | Via `tracing::info!` | Structured fields | stdout/stderr |
| **Runtime trace** | JSONL events | File-based | `state/runtime-trace.jsonl` |

**Prometheus metric names:**
- Counters: `agent_starts_total`, `llm_requests_total`, `tokens_input_total`, `tokens_output_total`, `tool_calls_total`, `channel_messages_total`, `heartbeat_ticks_total`, `errors_total`
- Histograms: `agent_duration_seconds`, `tool_duration_seconds`, `request_latency_seconds`
- Gauges: `tokens_used_last`, `active_sessions`, `queue_depth`

### 5.6 Health Monitoring

Global `HealthRegistry` singleton tracks named components:

```rust
ComponentHealth {
    status: "starting" | "ok" | "error",
    updated_at: RFC3339,
    last_ok: Option<RFC3339>,
    last_error: Option<String>,
    restart_count: u64,
}
```

Exposed via `/health` (public liveness) and `/api/health` (authenticated detail). Docker healthcheck polls `zeroclaw status` every 60 seconds.

### 5.7 Docker

| Stage | Base Image | Purpose | User |
|---|---|---|---|
| `builder` | `rust:1.93-slim` | Compile binary (BuildKit cached) | root |
| `dev` | `debian:trixie-slim` | Development runtime (has shell, curl) | 65534:65534 |
| `release` | `gcr.io/distroless/cc-debian13:nonroot` | Production (no shell, minimal surface) | 65534:65534 |

**Production compose:** resource limits 2 CPU / 2 GB; reservations 0.5 CPU / 512 MB; named volume `zeroclaw-data`; restart `unless-stopped`.

---

## 6. Configuration System

Config is loaded from `~/.zeroclaw/config.toml` (TOML format), with environment variable overrides and runtime API updates (`PUT /api/config`).

**Top-level sections (~40 subsections):**

```
[config]                    # api_key, default_provider, default_model, temperature
[agent]                     # max_tool_iterations, max_history_messages, parallel_tools
[autonomy]                  # level, workspace_only, allowed_commands, cost/action limits
[security]                  # sandbox, resources, audit, otp, estop
[security.sandbox]          # backend (auto/landlock/firejail/bubblewrap/docker/none)
[security.resources]        # max_memory_mb, max_cpu_time_seconds, max_subprocesses
[security.audit]            # enabled, log_path, max_size_mb, sign_events
[security.otp]              # enabled, method, gated_actions
[security.estop]            # enabled, state_file, require_otp_to_resume
[gateway]                   # port, host, pairing, rate limits, idempotency
[memory]                    # backend, embedding, vector/keyword weights, cache, hygiene
[storage]                   # provider, db_url, schema, table
[runtime]                   # kind (native/docker/wasm), docker config
[reliability]               # backoff settings
[scheduler]                 # (scheduler settings)
[heartbeat]                 # interval_minutes
[cron]                      # enabled
[channels.*]                # per-channel config (telegram, discord, slack, etc.)
[tunnel]                    # kind + per-tunnel config
[observability]             # backend, otel endpoint, runtime_trace
[cost]                      # daily_limit, monthly_limit, warn_threshold
[browser]                   # enabled, endpoint
[http_request]              # enabled
[web_search]                # enabled, provider, max_results
[multimodal]                # max_images, max_image_size_mb
[proxy]                     # HTTP/SOCKS proxy settings
[identity]                  # OpenClaw/AIEOS identity
[peripherals]               # board type, transport, baud rate
[hardware]                  # USB discovery settings
[hooks]                     # lifecycle hooks
[transcription]             # provider URL, model
[composio]                  # integration platform config
[agents.*]                  # delegate agent definitions
[model_routes]              # model routing rules
[embedding_routes]          # embedding routing rules
[query_classification]      # query classifier config
[skills]                    # user-defined skills config
[secrets]                   # encrypt (bool)
```

---

*Next file: `02_Data_Model.md` — Entity schemas, relationships, ER diagram, and state machines.*
