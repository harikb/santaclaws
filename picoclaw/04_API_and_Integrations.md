# 04 — API & Integrations

> Endpoints, agent tool catalog, channel adapters, and external service contracts.

---

## 1. HTTP API Surface

PicoClaw exposes a minimal HTTP surface. It is not a REST API — it is a webhook receiver and health probe.

### 1.1 Gateway Endpoints

| Method | Path | Purpose | Response |
|---|---|---|---|
| `GET` | `/health` | Liveness probe (Docker HEALTHCHECK) | `200` always — `{"status":"ok","uptime":"..."}` |
| `GET` | `/ready` | Readiness probe with pluggable checks | `200` if ready, `503` if not — `{"status":"ok/fail","uptime":"...","checks":{}}` |

### 1.2 Channel Webhook Endpoints

These are registered dynamically when a channel is enabled. Each runs on either the gateway port or its own configurable port.

| Channel | Method | Path | Port | Auth |
|---|---|---|---|---|
| **LINE** | `POST` | `/webhook/line` (configurable) | Configurable (`webhook_port`) | HMAC-SHA256 signature in `X-Line-Signature` header |
| **WeCom Bot** | `GET` | `/webhook/wecom` (configurable) | Configurable | SHA1 signature via query params (`msg_signature`, `timestamp`, `nonce`) |
| **WeCom Bot** | `POST` | `/webhook/wecom` (configurable) | Configurable | SHA1 signature + AES-CBC decryption |
| **WeCom App** | `GET` | `/webhook/wecom-app` (configurable) | Configurable | SHA1 signature via query params |
| **WeCom App** | `POST` | `/webhook/wecom-app` (configurable) | Configurable | SHA1 signature + AES-CBC decryption + `corp_id` verification |

**Non-HTTP channels** (no webhook endpoints — use platform-managed connections):

| Channel | Connection Type | Details |
|---|---|---|
| Telegram | Long-polling | 30-second poll timeout against Telegram Bot API |
| Discord | Gateway WebSocket | Discord gateway with heartbeat |
| Slack | Socket Mode WebSocket | Slack Events API via WebSocket |
| DingTalk | Stream Mode WebSocket | SDK-managed stream with auto-reconnect |
| OneBot | WebSocket client | Connects to configured `ws_url` with optional Bearer token |
| QQ | SDK-managed | QQ Bot framework handles connection |
| Feishu/Lark | SDK-managed | Lark SDK handles event subscription |
| WhatsApp | External bridge | Delegates to external WhatsApp bridge via `bridge_url` |
| MaixCAM | TCP server | Plain TCP socket, JSON protocol, not HTTP |

---

## 2. Agent Tool Catalog

Every tool is registered in the global `ToolRegistry` and exposed to the LLM as a function definition. The LLM calls tools by name with JSON arguments; the system executes them and returns results.

### 2.1 File Operations

#### `read_file(path) → content`

Read the contents of a file.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | `string` | Yes | Path to the file to read |

**Returns:** File content as text. Error if file not found or access denied.
**Workspace restriction:** If `restrict_to_workspace = true`, path must be within the agent's workspace.

---

#### `write_file(path, content) → confirmation`

Write content to a file, creating directories as needed.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | `string` | Yes | Path to the file to write |
| `content` | `string` | Yes | Content to write |

**Returns:** Silent confirmation. Creates parent directories with `0755`, writes file with `0644`.
**Workspace restriction:** Enforced.

---

#### `append_file(path, content) → confirmation`

Append content to the end of a file (creates if missing).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | `string` | Yes | File path to append to |
| `content` | `string` | Yes | Content to append |

**Returns:** Silent confirmation. Opens with `O_APPEND|O_CREATE|O_WRONLY`.
**Workspace restriction:** Enforced.

---

#### `list_dir(path) → listing`

List files and directories in a path.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | `string` | Yes | Directory path to list |

**Returns:** Formatted listing with `DIR:` and `FILE:` prefixes per entry.
**Workspace restriction:** Enforced.

---

#### `edit_file(path, old_text, new_text) → confirmation`

Replace exactly one occurrence of `old_text` with `new_text` in a file.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | `string` | Yes | File path to edit |
| `old_text` | `string` | Yes | Exact text to find |
| `new_text` | `string` | Yes | Replacement text |

**Returns:** Silent confirmation. Fails if `old_text` is not found or appears more than once (ambiguous edit).
**Workspace restriction:** Enforced.

---

### 2.2 Shell Execution

#### `exec(command, working_dir?) → output`

Execute a shell command and return its output.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `command` | `string` | Yes | Shell command to execute |
| `working_dir` | `string` | No | Working directory (defaults to workspace) |

**Returns:** Combined stdout/stderr. Truncated to 10,000 chars with notice if exceeded. Includes exit code on error.

**Runtime:** `sh -c` on Unix, `powershell` on Windows. 60-second timeout.

**Deny patterns** (regex, when `enable_deny_patterns = true`):

| Category | Blocked Patterns |
|---|---|
| Destructive | `rm -rf`, `del /f`, `format`, `dd` (disk writes) |
| System | `shutdown`, `reboot`, `chmod`, `chown` |
| Injection | `$()`, `${}`, backticks, `\| sh`, `\| bash` |
| Package managers | `npm install -g`, `pip install`, `apt`, `yum`, `dnf` |
| Network | `curl \| sh`, `wget \| sh`, `ssh` |
| Container | `docker run`, `docker exec` |
| VCS | `git push`, `git force` |
| Eval | `eval`, `source *.sh` |

**Workspace restriction:** If enabled, `working_dir` must be within workspace.

---

### 2.3 Web Tools

#### `web_search(query, count?) → results`

Search the web for current information.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | `string` | Yes | Search query |
| `count` | `integer` | No | Number of results, 1–10 |

**Returns:** Formatted list of titles, URLs, and snippets.

**Provider priority** (first enabled provider is used):

| Priority | Provider | Auth | Notes |
|---|---|---|---|
| 1 | Perplexity | API key | AI-enhanced results |
| 2 | Brave Search | API key | Standard web search |
| 3 | Tavily | API key | `search_depth: "advanced"` |
| 4 | DuckDuckGo | None | HTML scraping fallback, enabled by default |

---

#### `web_fetch(url, maxChars?) → content`

Fetch a URL and extract readable content.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `url` | `string` | Yes | URL to fetch (http/https only) |
| `maxChars` | `integer` | No | Max chars to extract (min 100, default 50,000) |

**Returns:** JSON object with `url`, `status`, `extractor`, `truncated`, `length`, `text`.

**Content handling:**
- JSON → formatted with `json.MarshalIndent`
- HTML → text extraction (scripts, styles, tags removed)
- Other → raw text

**Limits:** 60-second timeout. Up to 5 redirects followed.

---

### 2.4 Communication

#### `message(content, channel?, chat_id?) → confirmation`

Send a message to a user on a chat channel.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `content` | `string` | Yes | Message text |
| `channel` | `string` | No | Target channel (defaults to current context) |
| `chat_id` | `string` | No | Target chat/user ID (defaults to current context) |

**Returns:** Silent result. The tool tracks `sentInRound` to prevent the agent loop from sending a duplicate response.

**Interface:** Uses `SetContext(channel, chatID)` from `ContextualTool` — the agent loop injects the originating channel and chat before each turn.

---

### 2.5 Scheduling

#### `cron(action, ...) → result`

Schedule reminders, recurring tasks, or shell commands.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | `string` (enum) | Yes | `"add"`, `"list"`, `"remove"`, `"enable"`, `"disable"` |
| `message` | `string` | For `add` | Reminder/task message |
| `command` | `string` | No | Shell command to execute instead of agent turn |
| `at_seconds` | `integer` | No | One-time: seconds from now |
| `every_seconds` | `integer` | No | Recurring: interval in seconds |
| `cron_expr` | `string` | No | Cron expression (e.g., `"0 9 * * *"`) |
| `job_id` | `string` | For `remove`/`enable`/`disable` | Target job ID |
| `deliver` | `boolean` | No | If `true`, send result directly to channel. Default: `true`. Forced `false` for `command` payloads. |

**Returns:** For `add`: job ID and next run time. For `list`: formatted table of all jobs. For `remove`/`enable`/`disable`: confirmation.

**Schedule types:**
- `at_seconds` → `schedule.kind = "at"`, `schedule.atMs = now + seconds*1000`
- `every_seconds` → `schedule.kind = "every"`, `schedule.everyMs = seconds*1000`
- `cron_expr` → `schedule.kind = "cron"`, `schedule.expr = expr`

---

### 2.6 Agent Delegation

#### `spawn(task, label?, agent_id?) → async_result`

Spawn a background subagent for a task.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `task` | `string` | Yes | Task description for the subagent |
| `label` | `string` | No | Short display label |
| `agent_id` | `string` | No | Target agent ID (must be in parent's `allow_agents`) |

**Returns:** Async result — task is running in the background. Completion notification arrives via the `"system"` channel as an `InboundMessage`.

**Interface:** `AsyncTool` — receives a callback via `SetCallback()` for deferred result delivery.

**Permission check:** If `agent_id` is specified, `registry.CanSpawnSubagent(parentID, targetID)` is called. Requires `allow_agents: ["*"]` or explicit listing.

---

#### `subagent(task, label?) → result`

Execute a subagent task synchronously and return the result.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `task` | `string` | Yes | Task for the subagent |
| `label` | `string` | No | Short display label |

**Returns:** Dual output — `ForUser`: brief summary (max 500 chars), `ForLLM`: full execution details.

Runs `RunToolLoop()` inline — blocks until the subagent completes all tool iterations.

---

### 2.7 Skills Management

#### `find_skills(query, limit?) → search_results`

Search for installable skills from registries.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | `string` | Yes | Capability description (e.g., `"github integration"`) |
| `limit` | `integer` | No | Max results, 1–20 (default 5) |

**Returns:** Silent result with: count, slug, version, score, registry, display name, summary per result.

**Caching:** Normalized query checked against LRU cache (100 entries, 3600s TTL) with trigram similarity matching (Jaccard ≥ 0.7).

---

#### `install_skill(slug, registry, version?, force?) → confirmation`

Install a skill from a registry into the workspace.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `slug` | `string` | Yes | Skill identifier (e.g., `"weather"`) |
| `registry` | `string` | Yes | Registry name (e.g., `"clawhub"`) |
| `version` | `string` | No | Specific version (default: latest) |
| `force` | `boolean` | No | Overwrite if already installed (default: `false`) |

**Returns:** Silent result with success message, version, location. Warning prepended if `IsSuspicious`.

**Side effects:**
- Creates `{workspace}/skills/{slug}/` directory
- Writes `.skill-origin.json` metadata
- Skill available on next system prompt assembly

**Mutex:** Workspace-level lock prevents concurrent installations.

---

### 2.8 Hardware (Linux Only)

#### `i2c(action, bus?, address?, register?, data?, length?, confirm?) → result`

Interact with I2C bus devices.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | `string` (enum) | Yes | `"detect"`, `"scan"`, `"read"`, `"write"` |
| `bus` | `string` | For scan/read/write | Bus number (e.g., `"1"` for `/dev/i2c-1`) |
| `address` | `integer` | For read/write | 7-bit device address (`0x03`–`0x77`) |
| `register` | `integer` | No | Register address for targeted read/write |
| `data` | `integer[]` | For write | Bytes to write (0–255 each) |
| `length` | `integer` | No | Bytes to read (1–256, default 1) |
| `confirm` | `boolean` | For write | Safety guard — must be `true` to write |

**Returns:** JSON response. `detect`: list of available buses. `scan`: list of responding addresses. `read`: byte array. `write`: confirmation.

---

#### `spi(action, device?, speed?, mode?, bits?, data?, length?, confirm?) → result`

Interact with SPI bus devices.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `action` | `string` (enum) | Yes | `"list"`, `"transfer"`, `"read"` |
| `device` | `string` | For transfer/read | Device ID (e.g., `"2.0"` for `/dev/spidev2.0`) |
| `speed` | `integer` | No | Clock speed in Hz (default 1 MHz, max 125 MHz) |
| `mode` | `integer` | No | SPI mode 0–3 (CPOL/CPHA combinations, default 0) |
| `bits` | `integer` | No | Bits per word (1–32, default 8) |
| `data` | `integer[]` | For transfer | Bytes to send (0–255 each) |
| `length` | `integer` | For read | Bytes to read (1–4096) |
| `confirm` | `boolean` | For transfer | Safety guard — must be `true` to transfer |

**Returns:** JSON response. `list`: available SPI devices. `transfer`: received bytes (full-duplex). `read`: received bytes (sends zeros).

---

### 2.9 Tool Interface Summary

```
┌──────────────────────────────────────────────────────────────┐
│                      Tool Registry                            │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Base Tool   │  │ Contextual  │  │  Async Tool         │  │
│  │             │  │  Tool        │  │                     │  │
│  │ Name()      │  │ + SetContext │  │ + SetCallback       │  │
│  │ Description()│  │  (chan,chat) │  │   (AsyncCallback)   │  │
│  │ Parameters()│  │             │  │                     │  │
│  │ Execute()   │  │             │  │                     │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────────────┘  │
│         │                │                │                  │
│   read_file         message            spawn                │
│   write_file        cron                                    │
│   append_file                                               │
│   list_dir                                                  │
│   edit_file                                                 │
│   exec                                                      │
│   web_search                                                │
│   web_fetch                                                 │
│   subagent                                                  │
│   find_skills                                               │
│   install_skill                                             │
│   i2c                                                       │
│   spi                                                       │
└──────────────────────────────────────────────────────────────┘
```

**ToolResult structure** (returned by all tools):

| Field | Type | Description |
|---|---|---|
| `ForLLM` | `string` | Text fed back to the LLM as the tool output |
| `ForUI` | `string` | Optional text sent directly to the user (bypasses LLM) |
| `IsError` | `bool` | Whether the execution failed |
| `Async` | `bool` | Whether the result is still pending (spawn) |
| `Raw` | `any` | Internal data, not serialized to LLM |
| `Silent` | `bool` | Suppresses user-facing output from the agent loop |

---

## 3. LLM Provider Interface

All LLM backends implement a single interface:

```go
type LLMProvider interface {
    Chat(ctx, messages []Message, tools []ToolDefinition, model string, options map[string]any) (*LLMResponse, error)
    GetDefaultModel() string
}
```

### 3.1 Provider Types

| Type | Protocol | Examples |
|---|---|---|
| **OpenAI-compatible HTTP** | REST, `/v1/chat/completions` | OpenAI, OpenRouter, Groq, Zhipu, DeepSeek, Moonshot, Cerebras, Qwen, VolcEngine, Nvidia, ShengSuanYun, Ollama, vLLM, Gemini |
| **Anthropic HTTP** | REST, `/v1/messages` | Anthropic Claude |
| **AntiGravity** | OAuth2 + Gemini API | Google Cloud Code Assist |
| **Claude CLI** | Subprocess (`claude` binary) | Claude CLI |
| **Codex CLI** | Subprocess (`codex` binary) | GitHub Copilot Codex |
| **GitHub Copilot** | gRPC/stdio | GitHub Copilot |

### 3.2 Model Resolution

Models are identified by `{protocol}/{model-id}` (e.g., `openai/gpt-4o`, `anthropic/claude-sonnet-4-6`). Resolution order:

1. Check `model_list` for a matching `model_name` alias
2. Parse the protocol prefix to select provider type
3. If no prefix, default to OpenAI-compatible
4. Fall back to legacy `providers` config section

### 3.3 Fallback Chain

When multiple candidates are configured (`model.fallbacks`):

1. Try primary model
2. On retriable error (`rate_limit`, `timeout`, `overloaded`) → try next fallback
3. On non-retriable error (`auth`, `billing`, `format`) → fail immediately
4. Track per-model failures with cooldown (exponential backoff)

### 3.4 Chat Request/Response Contract

**Request** (assembled by agent loop):

| Field | Description |
|---|---|
| `messages` | Array of `{role, content, tool_calls?, tool_call_id?}` |
| `tools` | Array of `{type: "function", function: {name, description, parameters}}` |
| `model` | Model identifier string |
| `options` | Map with `max_tokens` (int) and `temperature` (float) |

**Response:**

| Field | Description |
|---|---|
| `content` | Text response from the model |
| `tool_calls` | Array of `{id, type, function: {name, arguments}}` — present when model wants to call tools |
| `finish_reason` | `"stop"` (natural end), `"length"` (token limit), `"tool_calls"` (tool invocation) |
| `usage` | `{prompt_tokens, completion_tokens, total_tokens}` |

---

## 4. Channel Adapter Contracts

Each channel implements the `Channel` interface:

```go
type Channel interface {
    Name() string
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Send(ctx context.Context, msg OutboundMessage) error
    IsRunning() bool
    IsAllowed(senderID string) bool
}
```

### 4.1 Channel Capabilities Matrix

| Channel | Connection | Voice | Media Download | Typing Indicator | Threading | Message Edit | Chunking |
|---|---|---|---|---|---|---|---|
| **Telegram** | Long-poll | Yes (Groq) | Photo, doc, audio, voice, video | No | No | Yes (placeholder) | No |
| **Discord** | WebSocket | Yes (Groq) | Image, video, file, record | Yes (continuous, 5 min max) | No | No | Yes (2000 char limit) |
| **Slack** | Socket Mode | Yes (Groq) | Files | No | Yes (thread replies) | No | No |
| **LINE** | HTTP webhook | No | Image, audio, video, file | Loading animation | No | No | No |
| **WeCom Bot** | HTTP webhook | No | Image, voice, file | No | No | No | No |
| **WeCom App** | HTTP webhook | No | XML-wrapped text | No | No | No | No |
| **DingTalk** | Stream WS | No | No | No | No | No | No |
| **QQ** | SDK-managed | No | No | No | No | No | No |
| **Feishu** | SDK-managed | No | No | No | No | No | No |
| **OneBot** | WebSocket | Yes (transcription) | Image, video, file, record | No | No | No | No |
| **WhatsApp** | External bridge | No | No | No | No | No | No |
| **MaixCAM** | TCP socket | No | Person detection | No | No | No | No |

### 4.2 Channel Authentication Details

| Channel | Auth Mechanism | Verification |
|---|---|---|
| **Telegram** | Bot token in API URL | N/A (outbound only) |
| **Discord** | Bot token in WebSocket handshake | N/A |
| **Slack** | Bot token + App token for Socket Mode | N/A |
| **LINE** | Channel access token (Bearer) | HMAC-SHA256 on request body with channel secret |
| **WeCom Bot** | Token + AES key | SHA1 signature on `(token, timestamp, nonce)` + AES-CBC decryption (PKCS7 padding) |
| **WeCom App** | Corp ID + Corp secret → access token | SHA1 signature + AES-CBC + Corp ID verification in decrypted payload |
| **DingTalk** | Client ID + Client secret | SDK-managed stream authentication |
| **QQ** | App ID + App secret | SDK-managed |
| **Feishu** | App ID + App secret | SDK-managed with encrypt key + verification token |
| **OneBot** | Optional Bearer access token | Token in WebSocket URL or header |
| **WhatsApp** | Bridge URL | Delegated to bridge |
| **MaixCAM** | None (TCP) | IP-based allow list |

### 4.3 Message Flow per Channel Type

**Polling channels** (Telegram):
```
Platform API ──poll──▶ Channel adapter ──publish──▶ Bus inbound
Bus outbound ──dispatch──▶ Channel adapter ──API call──▶ Platform API
```

**WebSocket channels** (Discord, Slack, DingTalk, OneBot):
```
Platform ──WS event──▶ Channel adapter ──publish──▶ Bus inbound
Bus outbound ──dispatch──▶ Channel adapter ──WS/API send──▶ Platform
```

**Webhook channels** (LINE, WeCom):
```
Platform ──HTTP POST──▶ Channel adapter ──verify──▶ ──publish──▶ Bus inbound
Bus outbound ──dispatch──▶ Channel adapter ──HTTP POST──▶ Platform API
```

**TCP channels** (MaixCAM):
```
Device ──TCP JSON──▶ Channel adapter ──publish──▶ Bus inbound
Bus outbound ──dispatch──▶ Channel adapter ──TCP JSON──▶ Device
```

---

## 5. External Service Integration Contracts

### 5.1 Search APIs

| Service | Endpoint | Auth | Request | Response Fields |
|---|---|---|---|---|
| **Brave Search** | `https://api.search.brave.com/res/v1/web/search` | `X-Subscription-Token` header | `?q={query}&count={count}` | `web.results[].{title, url, description}` |
| **Tavily** | Configurable `base_url` | API key in request body | `{query, search_depth: "advanced", max_results}` | `results[].{title, url, content}` |
| **DuckDuckGo** | `https://html.duckduckgo.com/html/` | None | Form POST `q={query}` | HTML scraping of result snippets |
| **Perplexity** | `https://api.perplexity.ai/chat/completions` | Bearer token | Chat completion format with search system prompt | `choices[].message.content` |

### 5.2 ClawHub Skills Registry

| Operation | Method | Endpoint | Auth |
|---|---|---|---|
| **Search** | `GET` | `{baseURL}/api/v1/search?q={query}&limit={n}` | Optional Bearer token |
| **Get metadata** | `GET` | `{baseURL}/api/v1/skills/{slug}` | Optional Bearer token |
| **Download** | `GET` | `{baseURL}/api/v1/download?slug={slug}&version={v}` | Optional Bearer token |

**Response size limits:** 2 MB for metadata/search, 50 MB for ZIP downloads.

**Moderation flags in metadata:** `is_malware_blocked` (hard block), `is_suspicious` (warning only).

### 5.3 Voice Transcription

| Service | Provider | Auth | Input | Output |
|---|---|---|---|---|
| **Groq** | Groq API | API key | Audio file (from Telegram/Discord/Slack voice messages) | Transcribed text |

Attached to channels at gateway startup. Channels that support voice (`SetTranscriber()`) convert audio to text before publishing to the bus.

### 5.4 OAuth Providers

| Provider | Flow | Token Storage |
|---|---|---|
| **Google Cloud Code Assist (AntiGravity)** | OAuth2 + PKCE | `~/.picoclaw/auth/tokens.json` |
| **GitHub Copilot** | Device code flow or OAuth | `~/.picoclaw/auth/tokens.json` |

Managed via `picoclaw auth login/logout/status` CLI commands.
