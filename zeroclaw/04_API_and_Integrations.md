# 04 — API & Agent Capabilities

> ZeroClaw Technical Blueprint | File 4 of 5

---

## 1. API Strategy

ZeroClaw exposes a single Axum HTTP/WebSocket server on port `42617`. The API follows a layered pattern:

- **Public routes** — no auth, health/metrics only
- **Pairing route** — one-time code exchange for bearer token
- **Webhook routes** — channel-specific ingestion (WhatsApp HMAC, Linq signing, etc.)
- **Authenticated API** — bearer token required, full management surface
- **Streaming** — SSE for events, WebSocket for interactive chat
- **Static** — embedded React SPA served at `/_app/*`

### Infrastructure Defaults

| Parameter | Value |
|---|---|
| Default port | `42617` |
| Default bind | `127.0.0.1` (localhost only) |
| Body size limit | 64 KB (global), 1 MB (`PUT /api/config`) |
| Request timeout | 30 seconds |
| Rate limit window | 60 seconds (sliding) |
| Max rate limit keys | 10,000 (LRU eviction) |
| Idempotency TTL | 300 seconds |
| SSE buffer | 256 events (ring buffer) |

---

## 2. Endpoint Catalog

### 2.1 Public Routes (No Auth)

#### `GET /health`

Liveness probe. Used by Docker healthcheck.

| Field | Value |
|---|---|
| Auth | None |
| Response | `{"status": "ok"}` |
| Status codes | 200 |

#### `GET /metrics`

Prometheus text exposition format.

| Field | Value |
|---|---|
| Auth | None |
| Response | Prometheus metrics text |
| Status codes | 200 |

---

### 2.2 Device Pairing

#### `POST /pair`

Exchange a one-time pairing code for a persistent bearer token.

| Field | Value |
|---|---|
| Auth | `X-Pairing-Code: <code>` header |
| Rate limit | 10/min per IP |
| Request body | None |
| Response (200) | `{"paired": true, "persisted": bool, "token": "...", "message": "..."}` |
| Response (403) | `{"error": "Invalid pairing code"}` |
| Response (429) | `{"error": "...", "retry_after": u64}` (lockout after repeated failures) |
| Side effects | Token persisted to `config.toml` |

---

### 2.3 Webhook Routes

#### `POST /webhook`

General-purpose message ingestion. Runs agent in simple mode (no tools).

| Field | Value |
|---|---|
| Auth | `Authorization: Bearer <token>` (if pairing required) + optional `X-Webhook-Secret` (SHA-256 HMAC) |
| Rate limit | 60/min per IP |
| Idempotency | Optional `X-Idempotency-Key` header (deduplicates within 300s TTL) |
| Request body | `{"message": "string"}` (max 64 KB) |
| Response (200) | `{"response": "string", "model": "string"}` |
| Response (200, duplicate) | `{"status": "duplicate", "idempotent": true, "message": "..."}` |
| Error responses | 400 (malformed), 401 (auth), 403 (HMAC), 429 (rate limit), 500 (agent error, sanitized) |

#### `GET /whatsapp`

Meta webhook verification (hub challenge).

| Field | Value |
|---|---|
| Auth | None |
| Query params | `hub.mode`, `hub.verify_token`, `hub.challenge` |
| Response | Echo `hub.challenge` value |

#### `POST /whatsapp`

WhatsApp Cloud API message webhook. Runs full agent loop WITH tools.

| Field | Value |
|---|---|
| Auth | `X-Hub-Signature-256` HMAC verification (constant-time) |
| Request body | WhatsApp Cloud API payload |
| Response | 200 (async processing, reply sent via WhatsApp channel) |

#### `POST /linq`

Linq (iMessage/RCS/SMS) webhook.

| Field | Value |
|---|---|
| Auth | Linq signing secret HMAC verification |
| Request body | Linq message payload |

#### `GET /wati` / `POST /wati`

WATI WhatsApp Business API verification and message webhook.

| Field | Value |
|---|---|
| Auth | WATI webhook HMAC verification |

#### `POST /nextcloud-talk`

Nextcloud Talk bot webhook.

| Field | Value |
|---|---|
| Auth | Nextcloud Talk bot token verification |

---

### 2.4 Authenticated Management API

All routes require `Authorization: Bearer <token>`. Returns 401 on invalid token.

#### `GET /api/status`

System overview.

```json
{
  "provider": "string",
  "model": "string",
  "temperature": 0.7,
  "uptime_seconds": 12345,
  "gateway_port": 42617,
  "locale": "string",
  "memory_backend": "sqlite",
  "paired": true,
  "channels": ["telegram", "discord"],
  "health": { /* HealthSnapshot */ }
}
```

#### `GET /api/config`

Current config as TOML. Sensitive fields (`api_key`, `bot_token`, `access_token`, `secret`, `app_secret`, `signing_secret`) replaced with `"[REDACTED]"`.

| Field | Value |
|---|---|
| Response | `{"format": "toml", "content": "..."}` |

#### `PUT /api/config`

Hot-reload config. Body limit: 1 MB.

| Field | Value |
|---|---|
| Request body | Raw TOML string |
| Response (200) | `{"status": "ok"}` |
| Response (400) | Invalid TOML |
| Side effects | Validates, writes to disk, hot-swaps runtime config |

#### `GET /api/tools`

List all registered tool specs.

```json
{
  "tools": [
    {"name": "shell", "description": "...", "parameters": { /* JSON Schema */ }}
  ]
}
```

#### `GET /api/cron`

List all scheduled jobs.

```json
{
  "jobs": [
    {"id": "uuid", "name": "...", "command": "...", "next_run": "rfc3339", "last_run": "rfc3339|null", "last_status": "ok|error|null", "enabled": true}
  ]
}
```

#### `POST /api/cron`

Create a new cron job.

| Field | Value |
|---|---|
| Request body | `{"name": "string?", "schedule": "string", "command": "string"}` |
| Response | `{"status": "ok", "job": {"id": "...", "name": "...", "command": "...", "enabled": true}}` |

#### `DELETE /api/cron/{id}`

Remove a cron job by ID.

| Field | Value |
|---|---|
| Response | `{"status": "ok"}` |

#### `GET /api/integrations`

List all integrations with their status.

```json
{
  "integrations": [
    {"name": "Telegram", "description": "...", "category": "chat", "status": "active"}
  ]
}
```

#### `POST /api/doctor`

Run diagnostics (model probing, connectivity checks).

```json
{
  "results": [/* diagnostic entries */],
  "summary": {"ok": 5, "warnings": 1, "errors": 0}
}
```

#### `GET /api/memory`

List or search memory entries.

| Field | Value |
|---|---|
| Query params | `query?: string`, `category?: string` |
| Behavior | If `query` present: recall (max 50). Otherwise: list all. |
| Response | `{"entries": [/* MemoryEntry[] */]}` |

#### `POST /api/memory`

Store a memory entry.

| Field | Value |
|---|---|
| Request body | `{"key": "string", "content": "string", "category": "string?"}` (default category: `"core"`) |
| Response | `{"status": "ok"}` |

#### `DELETE /api/memory/{key}`

Delete a memory entry.

| Field | Value |
|---|---|
| Response | `{"status": "ok", "deleted": true|false}` |

#### `GET /api/cost`

Cost tracking summary.

```json
{
  "cost": {
    "session_cost_usd": 0.15,
    "daily_cost_usd": 1.23,
    "monthly_cost_usd": 25.40,
    "total_tokens": 150000,
    "request_count": 42,
    "by_model": {
      "anthropic/claude-sonnet-4-20250514": {"cost_usd": 1.0, "total_tokens": 100000, "request_count": 30}
    }
  }
}
```

#### `GET /api/cli-tools`

List discovered CLI tools in the workspace.

#### `GET /api/health`

Detailed health snapshot (per-component status).

```json
{
  "health": {
    "pid": 12345,
    "updated_at": "rfc3339",
    "uptime_seconds": 3600,
    "components": {
      "gateway": {"status": "ok", "updated_at": "rfc3339", "restart_count": 0},
      "scheduler": {"status": "ok", "updated_at": "rfc3339", "restart_count": 0}
    }
  }
}
```

---

### 2.5 Streaming Endpoints

#### `GET /api/events` (SSE)

Server-Sent Events stream for real-time observability.

| Field | Value |
|---|---|
| Auth | `Authorization: Bearer <token>` |
| Protocol | `text/event-stream` |
| Buffer | 256 events (ring buffer; lagged clients silently skip) |

**Event types broadcast:**

| Event type | Fields |
|---|---|
| `agent_start` | `provider`, `model`, `timestamp` |
| `agent_end` | `provider`, `model`, `duration_ms`, `tokens_used`, `cost_usd`, `timestamp` |
| `llm_request` | `provider`, `model`, `timestamp` |
| `tool_call` | `tool`, `duration_ms`, `success`, `timestamp` |
| `tool_call_start` | `tool`, `timestamp` |
| `error` | `component`, `message`, `timestamp` |

#### `GET /ws/chat` (WebSocket)

Interactive agent chat over WebSocket.

| Field | Value |
|---|---|
| Auth | `?token=<bearer>` query parameter |
| Protocol | WebSocket (text frames, JSON) |

**Client → Server:**
```json
{"type": "message", "content": "user input text"}
```

**Server → Client:**

| Frame type | Fields | Notes |
|---|---|---|
| `done` | `full_response` | Successful completion |
| `error` | `message` | Parse error or agent failure |

> Note: `chunk`, `tool_call`, and `tool_result` frame types are defined but not yet implemented. Current mode is single-turn request/response.

---

## 3. Agent Tool Specifications

Tools are the execution surface exposed to the LLM. Each tool implements the `Tool` trait and provides a JSON Schema for parameter validation.

### 3.1 Default Tools (Always Registered)

#### `shell(command, approved?) -> ToolResult`

Execute a shell command in the workspace directory.

```
Inputs:
  command: string (required) — Shell command to execute
  approved: boolean (optional, default: false) — Explicit approval for medium/high-risk commands

Output:
  success: bool — Based on exit code
  output: string — stdout (max 1 MB, truncated)
  error: string? — stderr if non-empty

Security gates (in order):
  1. Rate limit check (20 actions/hour budget)
  2. Command allowlist validation
  3. Risk classification (Low/Medium/High)
  4. Forbidden path argument scan
  5. Action recording

Constraints:
  - Timeout: 60 seconds (kill_on_drop)
  - Output cap: 1 MB
  - Environment cleared; only PATH, HOME, TERM, LANG, LC_ALL, LC_CTYPE, USER, SHELL, TMPDIR + configured passthrough vars
  - Subshells blocked: $(), ``, ${}, <(), >()
  - Redirects blocked: >, <, >>
  - Background jobs blocked: bare &
  - Dangerous patterns blocked: find -exec, git config, chmod 777, curl|sh
```

#### `file_read(path, offset?, limit?) -> ToolResult`

Read file contents with line numbers.

```
Inputs:
  path: string (required) — Relative to workspace; absolute requires allowed_roots
  offset: integer (optional) — Starting line (1-based)
  limit: integer (optional) — Max lines to return

Output:
  output: "1: line content\n2: line content\n..." + "[Lines X-Y of Z]"

Constraints:
  - Max file size: 10 MB
  - Path traversal blocked (../, null bytes, symlink escapes)
  - Binary files: PDF extraction attempted, fallback to lossy UTF-8
  - Read-only: no action budget consumed
```

#### `file_write(path, content) -> ToolResult`

Write content to a file.

```
Inputs:
  path: string (required)
  content: string (required)

Output:
  "Written N bytes to {path}"

Security gates:
  - ReadOnly mode blocked
  - Path validation + symlink escape check (both directory and file)
  - Parent directories auto-created
  - Action budget consumed
```

#### `file_edit(path, old_string, new_string) -> ToolResult`

Replace an exact string match in a file.

```
Inputs:
  path: string (required)
  old_string: string (required) — Must appear exactly once
  new_string: string (required) — Empty string to delete

Output:
  "Edited {path}: replaced 1 occurrence (N bytes)"

Constraints:
  - old_string must not be empty
  - 0 matches → error ("not found")
  - 2+ matches → error ("ambiguous, appears N times")
  - Same security gates as file_write
```

#### `glob_search(pattern) -> ToolResult`

Search for files by glob pattern.

```
Inputs:
  pattern: string (required) — e.g., "**/*.rs", "src/**/mod.rs"

Output:
  Sorted workspace-relative paths, one per line, + "Total: N files"

Constraints:
  - Max results: 1,000
  - Absolute paths and ../ traversal rejected
  - Symlink escapes silently filtered
  - Directories excluded (files only)
```

#### `content_search(pattern, path?, output_mode?, include?, case_sensitive?, context_before?, context_after?, multiline?, max_results?) -> ToolResult`

Search file contents by regex.

```
Inputs:
  pattern: string (required) — Regex pattern
  path: string (default: ".")
  output_mode: "content" | "files_with_matches" | "count" (default: "content")
  include: string (optional) — File glob filter
  case_sensitive: boolean (default: true)
  context_before/after: integer (default: 0)
  multiline: boolean (default: false) — Requires ripgrep
  max_results: integer (default: 1000, hard cap: 1000)

Output:
  Matching content or file paths (workspace prefix stripped)

Constraints:
  - Timeout: 30 seconds
  - Output cap: 1 MB
  - Uses ripgrep when available; falls back to grep -rn -E
```

### 3.2 Memory Tools

#### `memory_store(key, content, category?) -> ToolResult`

```
Inputs:
  key: string (required) — Unique key for upsert
  content: string (required) — Text to store
  category: "core" | "daily" | "conversation" | custom string (default: "core")

Output: "Stored memory: {key}"
Security: Blocked in ReadOnly; rate-limited
```

#### `memory_recall(query, limit?) -> ToolResult`

```
Inputs:
  query: string (required) — Semantic search query
  limit: integer (default: 5)

Output: "Found N memories:\n- [category] key: content [score%]\n..."
Security: Read-only, no budget consumed
```

#### `memory_forget(key) -> ToolResult`

```
Inputs:
  key: string (required)

Output: "Forgot memory: {key}" or "No memory found with key: {key}"
Security: Blocked in ReadOnly; rate-limited
```

### 3.3 Cron Tools

#### `cron_add(schedule, command?, prompt?, name?, job_type?, ...) -> ToolResult`

```
Inputs:
  schedule: object (required) — {kind:"cron", expr, tz?} | {kind:"at", at} | {kind:"every", every_ms}
  command: string — Required for shell jobs
  prompt: string — Required for agent jobs
  name: string (optional)
  job_type: "shell" | "agent" (inferred if not set)
  session_target: "isolated" | "main" (default: "isolated")
  model: string (optional) — Override for agent jobs
  delivery: object (optional) — {mode, channel, to, best_effort}
  delete_after_run: boolean (default: true for At schedules)
  approved: boolean (default: false)

Output: JSON {id, name, job_type, schedule, next_run, enabled}
Security: ReadOnly blocked; command validated through allowlist
```

#### `cron_list() -> ToolResult`

```
Inputs: None
Output: JSON array of all jobs
Security: Read-only
```

#### `cron_remove(job_id) -> ToolResult`

```
Inputs: job_id: string (required)
Output: Success confirmation
Security: ReadOnly blocked; rate-limited
```

#### `cron_update(job_id, patch, approved?) -> ToolResult`

```
Inputs:
  job_id: string (required)
  patch: object (CronJobPatch — all fields optional)
  approved: boolean (default: false)

Output: Updated job object
Security: Command patches re-validated
```

#### `cron_run(job_id, approved?) -> ToolResult`

```
Inputs: job_id: string (required), approved: boolean (default: false)
Output: {job_id, status: "ok"|"error", duration_ms, output}
Security: Full execution security gates apply
```

#### `cron_runs(job_id, limit?) -> ToolResult`

```
Inputs: job_id: string (required), limit: integer (default: 10)
Output: Array of run records (output truncated to 500 chars each)
Security: Read-only
```

### 3.4 Conditional Tools

#### `browser(action, url?, selector?, value?, ...) -> ToolResult`

*Registered when `browser.enabled = true`*

```
Inputs:
  action: string (required) — "open", "snapshot", "click", "fill", "type", "get_text",
    "get_title", "get_url", "screenshot", "wait", "press", "hover", "scroll",
    "is_visible", "close", "find", "mouse_move", "mouse_click", "mouse_drag",
    "key_type", "key_press", "screen_capture"
  url: string — For "open" action
  selector: string — CSS selector, @ref, or text=...
  value/text/key: string — Action-specific values
  x, y: integer — Coordinate actions
  from_x, from_y, to_x, to_y: integer — Drag actions
  button: "left" | "right" | "middle"

Backends: agent_browser (default), rust-native (fantoccini), computer_use (CUA endpoint)
Security: Domain allowlist enforced; computer_use endpoint restricted to localhost by default
```

#### `http_request(url, method?, headers?, body?) -> ToolResult`

*Registered when `http_request.enabled = true`*

```
Inputs:
  url: string (required) — https:// or http:// only
  method: string (default: "GET") — GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
  headers: object (default: {})
  body: string (optional)

Output: Response body + status code
Constraints:
  - Domain allowlist enforced (http_request.allowed_domains)
  - Local/private hosts blocked
  - No redirects followed
  - Max response size: 1 MB (configurable)
  - Timeout: 30 seconds (configurable)
  - Sensitive headers redacted in logs
```

#### `web_search_tool(query) -> ToolResult`

*Registered when `web_search.enabled = true`*

```
Inputs:
  query: string (required)

Output: "Search results for: {query} (via {Provider})\n1. Title\n   URL\n   Snippet\n..."
Providers: duckduckgo (free, HTML scraping), brave (requires API key)
Constraints: Max results 1-10 (default: 5); timeout 15s
```

#### `delegate(agent, prompt, context?) -> ToolResult`

*Registered when `agents` map is non-empty*

```
Inputs:
  agent: string (required) — Name from configured agents map
  prompt: string (required) — Task description
  context: string (optional) — Prepended context

Output: Delegate agent's response text
Constraints:
  - Single-turn timeout: 120s; agentic (with tools) timeout: 300s
  - Depth tracking prevents infinite delegation chains
  - Agent name validated against config; unknown → error
```

### 3.5 Utility Tools

#### `git_operations(operation, ...) -> ToolResult`

```
Inputs:
  operation: "status" | "diff" | "log" | "branch" | "commit" | "add" | "checkout" | "stash"
  message: string — For commit
  paths/files: string — For add, diff
  branch: string — For checkout, branch
  cached: boolean — For diff --cached
  limit: integer — For log
  action: "push" | "pop" | "list" | "drop" — For stash
  index: integer — For stash drop

Read-only ops (no action budget): status, diff, log, branch
Write ops (require can_act): commit, add, checkout, stash
Security: Git argument injection blocked (--exec=, --upload-pack=, --no-verify, shell metacharacters)
```

#### `screenshot(filename?, region?) -> ToolResult`

```
Inputs:
  filename: string (optional, default: screenshot_<timestamp>.png)
  region: string (optional, macOS only: "selection" | "window")

Output: {path, base64} (PNG data)
Constraints: Timeout 15s; max 2 MB base64; filename sanitized
Platform: macOS (screencapture), Linux (gnome-screenshot → scrot → import)
```

#### `pdf_read(path, max_chars?) -> ToolResult`

```
Inputs:
  path: string (required)
  max_chars: integer (default: 50000, max: 200000)

Output: Extracted plain text
Constraints: Max PDF size 50 MB; requires rag-pdf feature
```

#### `image_info(path, include_base64?) -> ToolResult`

```
Inputs:
  path: string (required)
  include_base64: boolean (default: false)

Output: {format, width, height, size_bytes, base64?}
Constraints: Max 5 MB; detects PNG, JPEG, GIF, WebP, BMP from magic bytes
```

#### `pushover(message, title?, priority?, sound?) -> ToolResult`

```
Inputs:
  message: string (required)
  title: string (optional)
  priority: integer (-2 to 2, default: 0)
  sound: string (optional)

Output: Success confirmation
Credentials: PUSHOVER_TOKEN + PUSHOVER_USER_KEY from workspace .env file
```

---

## 4. Integration Interfaces

### 4.1 LLM Providers (30+)

All providers implement the `Provider` trait:

```
trait Provider: Send + Sync {
    fn name() -> &str
    fn default_model() -> &str
    async fn chat(messages, model, temperature, tools?) -> Result<ChatResponse>
    fn capabilities() -> ProviderCapabilities { native_tool_calling, vision }
}
```

**Input:** `Vec<ChatMessage { role, content }>` + optional tool specs

**Output:** `ChatResponse { text?, tool_calls: Vec<ToolCall>, usage?: TokenUsage, reasoning_content? }`

| Provider Tier | Examples | Tool Calling | Vision |
|---|---|---|---|
| Full-featured | Anthropic, OpenAI, Gemini | Native | Yes |
| OpenAI-compatible | OpenRouter, Groq, Together, Fireworks, DeepSeek, xAI, Mistral, Novita, Telnyx | Native | Varies |
| Regional | Qwen, GLM/Zhipu, MiniMax, Moonshot, Qianfan, Doubao | Native | Varies |
| Local | Ollama | Native (model-dependent) | Model-dependent |
| Cloud managed | Bedrock, Cloudflare AI, Vercel AI | Native | Varies |
| OAuth-based | OpenAI Codex, Gemini, Qwen OAuth, MiniMax OAuth | Native | Provider-dependent |

**Resilient wrapper** (`ReliableProvider`):
- Retries: `reliability.provider_retries` (default: 2) with `provider_backoff_ms` (default: 500ms)
- Fallback chain: `reliability.fallback_providers` (tried in order on primary failure)
- Model fallbacks: `reliability.model_fallbacks` (per-model alternative list)

**Credential resolution order:** Provider-specific env var → `ZEROCLAW_API_KEY` → `API_KEY`

### 4.2 Messaging Channels (20+)

All channels implement the `Channel` trait:

```
trait Channel: Send + Sync {
    fn name() -> &str
    async fn send(message: &SendMessage) -> Result<()>
    async fn listen(tx: mpsc::Sender<ChannelMessage>) -> Result<()>
    async fn health_check() -> bool
    async fn start_typing(recipient) -> Result<()>
    async fn stop_typing(recipient) -> Result<()>
    fn supports_draft_updates() -> bool
}
```

**Input (inbound):** `ChannelMessage { id, sender, reply_target, content, channel, timestamp, thread_ts? }`

**Output (outbound):** `SendMessage { content, recipient, subject?, thread_ts? }`

| Category | Channels | Auth Method |
|---|---|---|
| **Chat apps** | Telegram, Discord, Slack, Mattermost, DingTalk, QQ, Lark/Feishu | Bot token / WebSocket |
| **Messaging** | WhatsApp (Cloud + WATI), Signal, iMessage, Linq (iMessage/RCS/SMS) | API token / HMAC webhook |
| **Encrypted** | Matrix (E2EE), Nostr (NIP-04, NIP-59) | Device keys / keypair |
| **Legacy** | IRC (TLS), Email (SMTP+IMAP) | SASL / credentials |
| **Collaboration** | Nextcloud Talk | Bot webhook |
| **Voice** | ClawdTalk | — |
| **Local** | CLI | Process-local |

**Per-channel constants:**
- History per sender: 50 messages
- Message timeout: 300 seconds base
- Parallelism: 4 per channel
- Max in-flight: 8–64 (semaphore-controlled)
- Autosave threshold: 20+ characters
- Memory context injection: max 4 entries, max 4,000 chars

### 4.3 Tunnel Providers

| Provider | External Binary | Key Config |
|---|---|---|
| `cloudflare` | `cloudflared` | `tunnel.cloudflare.token` |
| `ngrok` | `ngrok` | `tunnel.ngrok.auth_token`, optional `domain` |
| `tailscale` | `tailscale` | `tunnel.tailscale.funnel` (bool), optional `hostname` |
| `custom` | User-specified | `tunnel.custom.start_command` with `{port}`, `{host}` placeholders |
| `none` | — | No tunnel (default) |

**Interface:** `start(host, port) -> public_url`, `stop()`, `health_check() -> bool`

### 4.4 Observability Backends

| Backend | Output | Endpoint |
|---|---|---|
| `log` | `tracing::info!` structured fields | stdout/stderr |
| `prometheus` | 11 metrics (counters, histograms, gauges) | `/metrics` |
| `otel` | OTLP metrics + spans | Configurable (default `localhost:4318`) |
| `noop` | Nothing | — |
| `multi` | Fan-out to multiple backends | — |
| `verbose` | Stderr progress indicators | stderr |

### 4.5 Sandbox Backends

| Backend | Platform | Mechanism |
|---|---|---|
| `landlock` | Linux 5.13+ | Kernel-level filesystem access control |
| `bubblewrap` | Linux | Userspace namespace isolation |
| `firejail` | Linux | Userspace sandbox with seccomp |
| `docker` | Any | Container isolation (alpine:3.20, network:none, read-only rootfs) |
| `auto` | — | Auto-detect best available |
| `none` | — | No sandboxing |

### 4.6 Hardware Peripherals

| Board | Transport | Key Capabilities |
|---|---|---|
| `nucleo-f401re` (STM32) | Serial / Probe | GPIO, ADC, memory read via debug probe |
| `rpi-gpio` (Raspberry Pi) | Native (`rppal`) | GPIO pin control, PWM |
| `arduino-uno` | Serial (UnoQ bridge) | GPIO, analog read |
| `esp32` | Serial / WebSocket | GPIO, WiFi, sensors |

**Peripheral tools exposed:** `hardware_board_info`, `hardware_memory_map`, `hardware_memory_read`

### 4.7 External Service Integrations

| Integration | Type | Config Key |
|---|---|---|
| **Composio** | Tool integration platform | `composio.api_key`, `composio.entity_id` |
| **Pushover** | Push notifications | `PUSHOVER_TOKEN`, `PUSHOVER_USER_KEY` (from .env) |
| **Web Search** | DuckDuckGo (free) / Brave (API key) | `web_search.provider`, `web_search.brave_api_key` |
| **Transcription** | Groq Whisper (default) | `transcription.api_url`, `transcription.model` |
| **Browser CUA** | Computer Use Agent endpoint | `browser.computer_use.endpoint` (default: localhost:8787) |

---

## 5. Tool Registration Summary

```
┌──────────────────────────────────────────────────────────┐
│                   Tool Registry                          │
├──────────────────────────────────────────────────────────┤
│ ALWAYS REGISTERED (6 default + 17 extended)              │
│                                                          │
│  Default:     shell, file_read, file_write, file_edit,   │
│               glob_search, content_search                │
│                                                          │
│  Extended:    cron_add, cron_list, cron_remove,           │
│               cron_update, cron_run, cron_runs,           │
│               memory_store, memory_recall, memory_forget, │
│               schedule, model_routing_config,             │
│               proxy_config, git_operations,               │
│               pushover, pdf_read, screenshot, image_info │
├──────────────────────────────────────────────────────────┤
│ CONDITIONAL                                              │
│                                                          │
│  browser.enabled=true     → browser_open, browser        │
│  http_request.enabled=true → http_request                │
│  web_search.enabled=true  → web_search_tool              │
│  composio.api_key set     → composio                     │
│  agents map non-empty     → delegate                     │
├──────────────────────────────────────────────────────────┤
│ PERIPHERAL (hardware feature + board configured)         │
│                                                          │
│  hardware_board_info, hardware_memory_map,               │
│  hardware_memory_read + board-specific GPIO tools        │
└──────────────────────────────────────────────────────────┘
```

---

*Next file: `05_Domain_Logic.md` — Business rules, validation constraints, edge cases, and technical debt.*
