# 01 — System Overview

## 1.1 What OpenClaw Is

OpenClaw is a **self-hosted, multi-channel AI gateway**. It bridges one or more AI language model agents with popular messaging platforms (Telegram, Discord, Slack, WhatsApp, Signal, iMessage, Matrix, MS Teams, and more). An operator installs the gateway on a machine — Mac, Linux, Raspberry Pi, cloud VM — and configures it so that an AI assistant is reachable from any connected messaging channel.

The system has three runtime surfaces:

| Surface | Technology | Purpose |
|---|---|---|
| **CLI** | TypeScript (Commander.js) on Node 22+ / Bun | Configuration, onboarding, diagnostics, one-shot agent commands |
| **Gateway Server** | Node.js HTTP + WebSocket (`ws`) | Persistent daemon that receives messages, runs agents, delivers replies |
| **Control UI** | Lit + @lit/signals (web) | Browser-based dashboard for status, sessions, chat, and config |

Native companion apps exist for **macOS** (SwiftUI), **iOS** (SwiftUI), and **Android** (Kotlin) — these connect to the gateway as paired devices.

---

## 1.2 Tech Stack Summary

### Runtime

| Layer | Choice | Notes |
|---|---|---|
| Language | TypeScript (strict, ESM) | Bun for dev/test execution; Node 22+ for production |
| CLI Framework | Commander.js | Lazy command registration; only loads needed modules |
| HTTP/WS Server | Node `http.createServer` + `ws` | Single port (default `18789`), HTTP + WS upgrade |
| Build | tsdown | Outputs to `dist/` |
| Test | Vitest + V8 coverage | 70% line/branch/function/statement thresholds |
| Lint/Format | Oxlint + Oxfmt | `pnpm check` / `pnpm format` |

### Data Storage

| Store | Engine | Location |
|---|---|---|
| Configuration | JSON5 file (Zod-validated) | `~/.openclaw/openclaw.json` |
| Session transcripts | JSONL files | `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl` |
| Session index | JSON file | `~/.openclaw/agents/<agentId>/sessions/sessions.json` |
| Vector memory | SQLite + sqlite-vec | Per-agent directory |
| Cron jobs | JSON file | `~/.openclaw/cron/` |
| Credentials | JSON files | `~/.openclaw/credentials/` |
| Paired devices | JSON file | Managed by device-pairing store |

There is **no central relational database**. All state is file-based, scoped per-agent or per-gateway instance.

### Frontend / Native Apps

| Platform | Stack |
|---|---|
| Web Control UI | Lit, @lit/signals, served from gateway HTTP |
| macOS app | SwiftUI, Sparkle auto-update, menubar gateway host |
| iOS app | SwiftUI, APNS push |
| Android app | Kotlin |

### Infrastructure & Networking

| Concern | Implementation |
|---|---|
| Deployment | Fly.io, Render, Docker, bare metal |
| Sandboxed execution | Docker containers (per-session, per-agent, or shared) |
| Secure networking | Tailscale (serve/funnel modes for external access) |
| LAN discovery | Bonjour/mDNS via `@homebridge/ciao` (service type `_openclaw-gw._tcp`) |
| Push notifications | APNS (iOS) |
| Webhook delivery | Google Pub/Sub (Gmail hooks) |
| macOS auto-update | Sparkle (`appcast.xml`) |

---

## 1.3 Gateway Server Architecture

```
┌──────────────────────────────────────────────────────────┐
│                   Gateway Process                        │
│                                                          │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────┐  │
│  │  HTTP Server │   │  WS Server   │   │ Channel      │  │
│  │             │──▶│  (ws)        │   │ Plugins      │  │
│  │  /v1/chat/* │   │              │   │ (grammy,     │  │
│  │  /hooks/*   │   │  req/res/evt │   │  discord.js, │  │
│  │  /api/*     │   │  frames      │   │  baileys...) │  │
│  │  / (UI)     │   │              │   │              │  │
│  └─────────────┘   └──────────────┘   └──────────────┘  │
│         │                  │                  │           │
│         ▼                  ▼                  ▼           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Command Queue (Lanes)                  │ │
│  │  main │ cron │ subagent │ nested                    │ │
│  └─────────────────────────────────────────────────────┘ │
│                        │                                 │
│                        ▼                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Agent Engine  │  │  Session     │  │  Outbound     │  │
│  │ (LLM call,   │  │  Store       │  │  Delivery     │  │
│  │  tools,      │  │  (JSONL)     │  │  (chunking,   │  │
│  │  sandbox)    │  │              │  │   streaming)  │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Bind Modes

The gateway binds to a configurable address:

| Mode | Binds To | Use Case |
|---|---|---|
| `loopback` (default) | `127.0.0.1` + `::1` | Local-only; apps on same machine |
| `lan` | `0.0.0.0` | Accessible on local network |
| `tailnet` | First Tailscale IP (`100.64.0.0/10`) | Accessible to Tailscale peers |
| `auto` | Prefers loopback, falls back to `0.0.0.0` | Automatic selection |
| `custom` | User-specified IP | Advanced setups |

### WebSocket Protocol

All gateway communication uses a JSON frame protocol over WebSocket:

| Frame Type | Direction | Structure |
|---|---|---|
| `req` | Client → Server | `{ type: "req", id, method, params? }` |
| `res` | Server → Client | `{ type: "res", id, ok, payload?, error? }` |
| `event` | Server → Client | `{ type: "event", event, payload?, seq? }` |

On connect, the client sends `ConnectParams` (protocol version, client identity, auth credentials). The server replies with `HelloOk` containing a full state snapshot, supported methods/events, and rate-limit policy.

### Command Queue (Process Lanes)

Work is serialized through an in-process command queue organized into **lanes**:

| Lane | Purpose | Concurrency |
|---|---|---|
| `main` | Primary inbound message → agent turn pipeline | 1 (serial) |
| `cron` | Scheduled job execution | 1 (serial) |
| `subagent` | Spawned sub-agent turns | Configurable |
| `nested` | Nested agent calls | Configurable |

Each lane is independent. Tasks within a lane execute serially by default. A generation counter handles in-process restarts (SIGUSR1) by invalidating stale task completions. Graceful shutdown polls active tasks at 50ms intervals.

---

## 1.4 Roles and Permissions

### Gateway Roles

There are exactly **two** top-level roles:

| Role | Description |
|---|---|
| `operator` | The gateway owner. Full administrative control. |
| `node` | A remote machine (browser, exec host). Limited to node-specific methods. |

### Operator Scopes

Operator permissions are further divided into five scopes:

| Scope | Grants |
|---|---|
| `operator.admin` | **Implies all other scopes.** Agent CRUD, channel logout, session management, config changes, cron management, skill install, system events, update control |
| `operator.write` | **Implies `operator.read`.** Send messages, run agent turns, wake, talk mode, TTS, node invoke, browser requests, push tests |
| `operator.read` | Health, status, logs, channel status, model list, tool catalog, agent list, session list/preview, cron list, usage/cost queries, config read |
| `operator.approvals` | Approve/deny exec requests (shell command execution approval flow) |
| `operator.pairing` | Manage node and device pairing: list, approve, reject, remove, token rotate/revoke |

**Scope implication hierarchy:**

```
operator.admin ──▶ operator.read
                ──▶ operator.write ──▶ operator.read
                ──▶ operator.approvals
                ──▶ operator.pairing
```

The CLI grants **all five scopes** by default. Paired devices receive only the scopes approved at pairing time.

### Permissions Matrix

| Action | operator.admin | operator.write | operator.read | operator.approvals | operator.pairing | node |
|---|---|---|---|---|---|---|
| View status/health | Yes | Yes | Yes | — | — | — |
| Read sessions/logs | Yes | Yes | Yes | — | — | — |
| Send messages | Yes | Yes | — | — | — | — |
| Run agent turns | Yes | Yes | — | — | — | — |
| Approve exec requests | Yes | — | — | Yes | — | — |
| Manage pairing | Yes | — | — | — | Yes | — |
| CRUD agents/config | Yes | — | — | — | — | — |
| Manage cron jobs | Yes | — | — | — | — | — |
| Install skills | Yes | — | — | — | — | — |
| Node invoke result | — | — | — | — | — | Yes |
| Node event push | — | — | — | — | — | Yes |
| `health` method | Yes | Yes | Yes | Yes | Yes | Yes |

### Channel-Level Access Control

Beyond gateway roles, **inbound message senders** are gated by per-channel policies:

| Policy | Type | Values | Effect |
|---|---|---|---|
| **DM Policy** | `DmPolicy` | `pairing` (default), `allowlist`, `open`, `disabled` | Controls who can DM the bot |
| **Group Policy** | `GroupPolicy` | `open`, `allowlist` (default), `disabled` | Controls which groups the bot responds in |

**DM Policy behavior:**

| Value | Unknown Sender | Known/Allowed Sender |
|---|---|---|
| `pairing` | Receives 8-char pairing code; queued until operator approves | Allowed |
| `allowlist` | Silently blocked | Allowed |
| `open` | Allowed | Allowed |
| `disabled` | Blocked | Blocked |

### Effective User Role Hierarchy (Inbound Senders)

From most to least privileged:

```
Operator (local CLI / full-scope paired device)
  └─▶ Paired Device (scoped — approved scopes only)
       └─▶ Paired Sender (completed channel pairing handshake)
            └─▶ Allowed Sender (on per-channel allowlist)
                 └─▶ Group Member (in allowed group/guild)
                      └─▶ Open Sender (any user, when DM policy = "open")
```

---

## 1.5 Authentication Mechanisms

### Gateway Auth Modes

| Mode | Mechanism | Use Case |
|---|---|---|
| `none` | No authentication | Local-only / trusted network |
| `token` | Static bearer token (`OPENCLAW_GATEWAY_TOKEN` or config) | Remote access with shared secret |
| `password` | Password auth (disables Tailscale auto-auth) | Simple remote access |
| `trusted-proxy` | Reverse proxy injects user identity via headers | Behind nginx/Caddy/Cloudflare |
| Tailscale (layered) | Verified via Tailscale whois API | Automatic auth for Tailscale peers |

### Device Pairing Protocol

Paired devices (mobile apps, remote Control UI) use a cryptographic handshake:

1. **Device generates** an Ed25519 keypair and presents `deviceId + publicKey + role + requestedScopes` to gateway.
2. **Operator approves** via `device.pair.approve` (or auto-approved if request is from localhost).
3. **Gateway issues** a per-role device token bound to approved scopes.
4. **On each connect**, device signs a payload: `v2|deviceId|clientId|clientMode|role|scopes|timestampMs|token|nonce` with its private key. Gateway verifies using stored public key. Nonce is server-generated per connection (replay protection). Timestamp skew limit: ±2 minutes.
5. **Tokens are revocable** (`device.token.revoke`) and **rotatable** (`device.token.rotate`).
6. **Scope upgrades** trigger re-pairing approval.

### Auth Rate Limiting

Brute-force protection is applied per-IP at two scopes:
- `AUTH_RATE_LIMIT_SCOPE_SHARED_SECRET` — for token/password attempts
- `AUTH_RATE_LIMIT_SCOPE_DEVICE_TOKEN` — for device token attempts

Control-plane write operations (`config.apply`, `config.patch`, `update.run`) are rate-limited to 3 writes per 60 seconds per client.

---

## 1.6 Plugin System

OpenClaw is extensible via a plugin architecture. Plugins can provide:
- **Channel integrations** (new messaging platforms)
- **Memory backends** (alternative vector stores)
- **Agent tools** (new capabilities)
- **CLI commands** (additional subcommands)

### Plugin Discovery Order

1. **Config-specified** — paths in `plugins.load` config key
2. **Workspace** — `<workspaceDir>/.openclaw/extensions/`
3. **Global** — `~/.openclaw/extensions/`
4. **Bundled** — shipped with the `openclaw` npm package

### Security Checks (Pre-Load)

| Check | Rejects If |
|---|---|
| Path escape | Resolved path escapes plugin root |
| Broken symlink | `stat()` fails |
| World-writable | Mode bit `0o002` set |
| Ownership (non-bundled) | UID does not match process UID or root |

### Loading Mechanism

- Plugins are loaded via `jiti` (TypeScript/ESM runtime loader).
- A plugin's `package.json` declares `openclaw.extensions` metadata.
- The module's default export is either a `register(api)` function or an object with `register`/`activate`.
- Registration is **synchronous** (async registration is warned and ignored).
- Plugin config is validated against a JSON Schema declared in the plugin's `package.json`.
- Only **one** memory-backend plugin can be active at a time (controlled by `plugins.slots.memory`).

### Plugin Registry Cache

Loaded plugin registries are cached in-memory keyed by `workspaceDir + JSON.stringify(pluginsConfig)`. The cache persists for the process lifetime.

---

## 1.7 Infrastructure Services

### Bonjour/mDNS (LAN Discovery)

- Service type: `_openclaw-gw._tcp` on domain `local`
- TXT record includes: `role=gateway`, port, LAN host, display name, optional TLS fingerprint, Tailscale DNS, CLI path
- Watchdog re-advertises at 60s intervals if service state drifts (handles sleep/wake)
- Disabled in test environments or via `OPENCLAW_DISABLE_BONJOUR=1`

### Tailscale Integration

- Detects Tailscale interfaces by scanning for IPs in `100.64.0.0/10` (IPv4) or `fd7a:115c:a1e0::/48` (IPv6)
- Bind mode `tailnet` binds directly to the Tailscale IPv4 address
- `serve`/`funnel` modes expose the gateway through Tailscale's reverse proxy for external HTTPS access

### Docker Sandbox

- Containers run agent exec commands in isolation
- Scoping: `session` (per-session), `agent` (per-agent), or `shared` (one for all)
- Security hardening: `--security-opt no-new-privileges`, capability drops, optional read-only root FS, seccomp/AppArmor profiles
- Env var sanitization blocks secrets from leaking into sandbox
- Config hash label (`openclaw.configHash`) triggers container recreation when sandbox config changes
- Hot container protection: containers used in the last 5 minutes are not replaced

### Session Store Concurrency

- Per-path in-process lock queue serializes concurrent writes
- Filesystem-level lock (`acquireSessionWriteLock()`) guards against multi-process access
- Atomic writes via temp file + `rename()` (mode `0o600` on POSIX)
- 45-second TTL cache with mtime-based invalidation prevents stale reads

### Session Maintenance

| Parameter | Default |
|---|---|
| Max age | 30 days |
| Max entries | 500 |
| Max file size (rotation) | 10 MB |

---

## 1.8 Key Environment Variables

| Variable | Purpose |
|---|---|
| `OPENCLAW_STATE_DIR` | Override state directory (`~/.openclaw`) |
| `OPENCLAW_CONFIG_PATH` | Override config file path |
| `OPENCLAW_GATEWAY_PORT` | Override gateway port (default `18789`) |
| `OPENCLAW_GATEWAY_TOKEN` | Gateway auth token |
| `OPENCLAW_DISABLE_BONJOUR` | Disable mDNS advertising |
| `OPENCLAW_SESSION_CACHE_TTL_MS` | Override session cache TTL |
| `ANTHROPIC_API_KEY` | Anthropic Claude API key |
| `OPENAI_API_KEY` | OpenAI API key |
| `GOOGLE_API_KEY` | Google Gemini API key |
| `ELEVENLABS_API_KEY` | ElevenLabs TTS key |
| `BRAVE_API_KEY` | Brave Search key |
| `DISCORD_BOT_TOKEN` | Discord bot token |
| `SLACK_APP_TOKEN` / `SLACK_BOT_TOKEN` | Slack tokens |
