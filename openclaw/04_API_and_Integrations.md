# 04 — API & Integrations

## 4.1 API Strategy

OpenClaw exposes three API surfaces from a single gateway process on one port (default `18789`):

| Surface | Transport | Auth | Purpose |
|---|---|---|---|
| **WebSocket RPC** | JSON frames over WS | Device token, shared token, password, Tailscale whois, trusted-proxy | Real-time bidirectional: all gateway methods, events, streaming |
| **HTTP REST** | Standard HTTP | Bearer token (`Authorization` or `X-OpenClaw-Token`) | Webhooks, OpenAI-compatible API, tool invocation, Control UI |
| **CLI** | Local process → WS | Inherits all operator scopes | User-facing commands that proxy to gateway methods |

All three surfaces ultimately dispatch to the same internal method handlers.

---

## 4.2 HTTP Endpoints

### OpenAI-Compatible Chat Completions

```
POST /v1/chat/completions
Auth: Bearer <gateway-token>
Max body: 1 MB
```

| Input Field | Type | Notes |
|---|---|---|
| `model` | `string?` | Resolves to agent ID; defaults to `"openclaw"` |
| `messages` | `Message[]` | `system`/`developer`/`user`/`assistant`/`tool` roles |
| `stream` | `boolean?` | SSE streaming if true |
| `user` | `string?` | Scopes session key per user |

**Behavior:** System/developer messages become `extraSystemPrompt`. Conversation is condensed and dispatched to internal `agentCommand()`. Session key: `openai:<ip>:<agentId>:<user>`.

**Non-streaming response:** Standard `chat.completion` JSON with `choices[0].message.content`.
**Streaming response:** SSE with `chat.completion.chunk` deltas, terminated by `data: [DONE]`.

---

### OpenResponses API

```
POST /v1/responses
Auth: Bearer <gateway-token>
Max body: 20 MB
```

| Input Field | Type | Notes |
|---|---|---|
| `model` | `string` | Required; resolves to agent ID |
| `input` | `string \| InputItem[]` | Messages, function calls/outputs, reasoning, item references |
| `instructions` | `string?` | System-level instructions |
| `tools` | `FunctionTool[]?` | Client-defined function tools |
| `tool_choice` | `string \| object?` | `auto` / `none` / `required` / specific function |
| `stream` | `boolean?` | SSE streaming |
| `max_output_tokens` | `number?` | Output token cap |
| `reasoning` | `{ effort?, summary? }?` | Reasoning control |
| `user` | `string?` | Session scoping |

**Streaming events (in order):**
```
response.created → response.in_progress → response.output_item.added →
response.content_part.added → response.output_text.delta (repeated) →
response.output_text.done → response.content_part.done →
response.output_item.done → response.completed → [DONE]
```

If the agent calls a client-defined tool: `status: "incomplete"`, output contains a `function_call` item for the client to fulfill.

---

### Inbound Webhooks

```
POST {hooks.basePath}/*
Auth: Bearer <hooks.token> (via Authorization or X-OpenClaw-Token header)
Max body: 256 KB (configurable)
Rate limit: 20 failures / 60s → 429
```

| Endpoint | Purpose | Response |
|---|---|---|
| `POST /hooks/wake` | Trigger heartbeat. Body: `{ text, mode? }` | `200 { ok, mode }` |
| `POST /hooks/agent` | Run agent turn. Body: `{ message, agentId?, sessionKey?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }` | `202 { ok, runId }` |
| `POST /hooks/{mapping-path}` | Custom mapped webhook (from `hooks.mappings[]`) | `202` or `200` or `204` |

Query-param tokens are explicitly rejected (400). Token in `?token=` returns an error directing callers to use the header.

---

### Tool Invocation

```
POST /tools/invoke
Auth: Bearer <gateway-token>
Max body: 2 MB
```

```json
{
  "tool": "web_search",
  "action": "search",
  "args": { "query": "latest news", "count": 5 },
  "sessionKey": "agent:main:main",
  "dryRun": false
}
```

Optional headers: `X-OpenClaw-Message-Channel`, `X-OpenClaw-Account-Id`.
Response: `{ ok: true, result: <tool-output> }`. Subject to full tool policy pipeline.

---

### Channel Plugin Routes

```
ANY /api/channels/*     — Gateway-auth-protected channel plugin HTTP endpoints
ANY <plugin-defined>    — Plugin-owned routes (plugin manages its own auth)
POST /slack/events      — Slack Event API callbacks (per-account configurable path)
```

---

### Control UI & Canvas

| Path | Purpose | Auth |
|---|---|---|
| `GET {basePath}/` | Serve Control UI SPA | None (CSP restricted) |
| `GET {basePath}/__openclaw/control-ui-config.json` | Bootstrap config | None |
| `GET {basePath}/avatar/{agentId}` | Agent avatar image | None |
| `GET /__openclaw__/a2ui/*` | A2UI SPA assets | Canvas capability token |
| `WS /__openclaw__/ws` | Canvas WebSocket | Canvas capability token |

---

## 4.3 WebSocket Methods (Complete Reference)

### Read Methods (`operator.read`)

| Method | Params | Returns | Purpose |
|---|---|---|---|
| `health` | — | `{ ok, uptime, version }` | Gateway health (also unauthenticated) |
| `status` | — | System status summary | Overall system state |
| `channels.status` | — | `ChannelAccountSnapshot[]` | Per-channel connection state |
| `models.list` | — | `ModelDefinition[]` | Available AI models |
| `tools.catalog` | `{ agentId?, includePlugins? }` | `{ profiles, groups[] }` | Available agent tools by profile |
| `agents.list` | — | `AgentConfig[]` | Configured agents |
| `agent.identity.get` | `{ agentId }` | `IdentityConfig` | Agent name, avatar, theme |
| `sessions.list` | — | `SessionEntry[]` | Stored sessions |
| `sessions.preview` | `{ sessionKey }` | Session content preview | Recent messages |
| `sessions.resolve` | `{ sessionKey }` | Resolved session info | Session lookup |
| `sessions.usage` | — | Usage statistics | Token counts, costs |
| `sessions.usage.timeseries` | `{ range }` | Time-series usage | Historical data |
| `sessions.usage.logs` | — | Usage log entries | Per-turn logs |
| `cron.list` | — | `CronJob[]` | Scheduled jobs |
| `cron.status` | — | Scheduler state | Running/idle |
| `cron.runs` | `{ jobId? }` | `CronRunLogEntry[]` | Execution history |
| `logs.tail` | `{ lines? }` | Log stream | Gateway file logs |
| `usage.status` | — | Usage metrics | Aggregate metrics |
| `usage.cost` | — | Cost data | Token cost breakdown |
| `skills.status` | — | Installed skills | Skill inventory |
| `config.get` | `{ key }` | Config value | Read config |
| `node.list` | — | Paired nodes | Node inventory |
| `node.describe` | `{ nodeId }` | Node details | Node info |
| `chat.history` | `{ sessionKey }` | Message history | Webchat history |
| `agents.files.list` | `{ agentId }` | File list | Agent workspace files |
| `agents.files.get` | `{ agentId, path }` | File content | Read workspace file |
| `tts.status` | — | TTS state | Provider status |
| `tts.providers` | — | Provider list | Available TTS providers |
| `voicewake.get` | — | Voice wake config | Current settings |
| `talk.config` | — | Talk mode config | Voice settings |
| `system-presence` | — | Presence state | Current system presence |
| `last-heartbeat` | — | Timestamp | Last heartbeat time |

### Write Methods (`operator.write`)

| Method | Params | Returns | Purpose |
|---|---|---|---|
| `send` | `{ channel, to, text, media? }` | `{ delivered }` | Send message to channel |
| `agent` | `{ message, agentId?, sessionKey?, model?, thinking? }` | `{ runId }` | Run agent turn (fire-and-forget) |
| `agent.wait` | Same as `agent` | `{ reply, usage }` | Run agent turn and wait for result |
| `wake` | `{ text?, mode? }` | `{ ok }` | Trigger heartbeat |
| `poll` | `{ sessionKey }` | Pending messages | Poll for queued messages |
| `chat.send` | `{ text, sessionKey? }` | `{ ok }` | Send webchat message |
| `chat.abort` | `{ sessionKey }` | `{ ok }` | Abort in-progress agent |
| `talk.mode` | `{ mode }` | `{ ok }` | Set talk/voice mode |
| `tts.enable` | — | `{ ok }` | Enable TTS |
| `tts.disable` | — | `{ ok }` | Disable TTS |
| `tts.convert` | `{ text }` | Audio data | Text-to-speech |
| `tts.setProvider` | `{ provider }` | `{ ok }` | Set TTS provider |
| `voicewake.set` | `{ config }` | `{ ok }` | Set voice wake config |
| `node.invoke` | `{ nodeId, command, args }` | `{ result }` | Run command on remote node |
| `browser.request` | `{ action, params }` | `{ result }` | Browser automation command |
| `push.test` | — | `{ ok }` | Send test push notification |

### Admin Methods (`operator.admin`)

| Method | Purpose |
|---|---|
| `agents.create` / `agents.update` / `agents.delete` | Agent CRUD |
| `channels.logout` | Disconnect a channel |
| `sessions.patch` / `sessions.reset` / `sessions.delete` / `sessions.compact` | Session management |
| `cron.add` / `cron.update` / `cron.remove` / `cron.run` | Cron job management |
| `skills.install` / `skills.update` | Skill management |
| `connect` | Establish/re-authenticate WS connection |
| `chat.inject` | Inject message into transcript |
| `set-heartbeats` | Configure heartbeat schedule |
| `system-event` | Emit system event |
| `agents.files.set` | Write agent workspace file |
| `web.login.start` / `web.login.wait` | Web provider OAuth flow |
| `config.*` (prefix) | Config apply, patch, etc. (rate-limited: 3/60s) |
| `wizard.*` (prefix) | Onboarding wizard methods |
| `update.*` (prefix) | Update management (rate-limited: 3/60s) |

### Approval Methods (`operator.approvals`)

| Method | Purpose |
|---|---|
| `exec.approval.request` | Register exec approval request |
| `exec.approval.waitDecision` | Wait for operator decision |
| `exec.approval.resolve` | Approve or deny exec |

### Pairing Methods (`operator.pairing`)

| Method | Purpose |
|---|---|
| `node.pair.request` / `list` / `approve` / `reject` / `verify` | Node pairing lifecycle |
| `device.pair.list` / `approve` / `reject` / `remove` | Device pairing lifecycle |
| `device.token.rotate` / `revoke` | Device token management |
| `node.rename` | Rename a paired node |

### Node-Role Methods (node clients only)

| Method | Purpose |
|---|---|
| `node.invoke.result` | Report invocation result to gateway |
| `node.event` | Push event to gateway |
| `skills.bins` | Report installed skill binaries |

---

## 4.4 Agent Tools as Callable Functions

These are the tools available to the AI agent during a turn. Each tool can be thought of as a function callable by an AI agent.

### File System Tools

```
read(path: string, offset?: number, limit?: number) → string
  Read file contents. Supports line ranges.

write(path: string, content: string) → { ok: boolean }
  Create or overwrite a file.

edit(path: string, old_text: string, new_text: string) → { ok: boolean }
  Precise text replacement within a file.

apply_patch(path: string, patch: string) → { ok: boolean }
  Apply a unified diff patch to a file.
```

### Runtime Tools

```
exec(command: string, timeout_ms?: number, cwd?: string) → { stdout, stderr, exit_code }
  Execute a shell command. Subject to exec security policy (deny/allowlist/full).
  May trigger approval flow if exec policy requires it.

process(action: "list" | "start" | "stop" | "signal", pid?: number, command?: string) → ProcessResult
  Manage background processes.
```

### Web Tools

```
web_search(query: string, count?: number, freshness?: string) → SearchResult[]
  Search the web via Brave/Perplexity/Grok.
  Returns: [{ title, url, description, age }]

web_fetch(url: string, max_chars?: number) → { content, extractor, content_type }
  Fetch and extract web page content. Falls back to Firecrawl on failure.
```

### Memory Tools

```
memory_search(query: string, limit?: number) → MemorySearchResult[]
  Semantic search over agent memory files and session transcripts.
  Returns: [{ path, startLine, endLine, score, snippet, source, citation }]

memory_get(path: string) → string
  Read a specific memory file.
```

### Session Tools

```
sessions_list(filter?: { agentId?, channel?, status? }) → SessionSummary[]
  List conversation sessions with metadata.

sessions_history(session_key: string, limit?: number) → Message[]
  Retrieve conversation history for a session.

sessions_send(session_key: string, message: string) → { ok: boolean }
  Send a message to another session (agent-to-agent messaging).

sessions_spawn(task: string, model?: string, agentId?: string) → { sessionKey, runId }
  Spawn a sub-agent with its own session. Max depth and child count limits apply.

subagents(action: "list" | "kill", runId?: string) → SubagentInfo[]
  List or kill spawned sub-agents.

session_status() → { sessionKey, agentId, model, tokens, compactionCount }
  Current session metadata.
```

### Messaging Tools

```
message(channel: string, to: string, text: string, media?: string) → { delivered }
  Send a message to a specific channel and recipient.
```

### Browser Tools

```
browser(action: string, params: object) → BrowserResult
  Control a web browser. Actions: navigate, click, type, scroll,
  screenshot, evaluate, snapshot, wait, download, form_fill.
```

### Media Tools

```
image(path: string, prompt?: string) → { description: string }
  Understand/describe an image using vision models.

tts(text: string, voice?: string) → { audioPath: string }
  Convert text to speech audio file.
```

### Automation Tools

```
cron(action: "list" | "add" | "update" | "remove" | "run", job?: CronJobSpec) → CronResult
  Manage scheduled tasks.

gateway(action: string, params?: object) → GatewayResult
  Gateway control operations.
```

### Tool Profiles

Tools are gated by profiles. The active profile determines the default tool set:

| Profile | Tools Included |
|---|---|
| `minimal` | `session_status` only |
| `coding` | FS tools, exec, process, memory, sessions (all), image |
| `messaging` | sessions (list, history, send), session_status, message |
| `full` | All tools (no restrictions) |

Additional tools can be enabled/disabled via `tools.allow[]`, `tools.alsoAllow[]`, `tools.deny[]` in config. Provider-level and group-level overrides are supported.

---

## 4.5 External Integrations

### LLM Providers

| Provider | API Adapter | Auth | Default Model |
|---|---|---|---|
| Anthropic | `anthropic-messages` | API key | `claude-opus-4-6` |
| OpenAI | `openai-completions` | API key | `gpt-5-mini` |
| Google Gemini | `google-generative-ai` | API key | `gemini-3-flash-preview` |
| AWS Bedrock | `bedrock-converse-stream` | AWS SDK (IAM) | Auto-discovered |
| GitHub Copilot | Built-in OAuth | OAuth token | Exchange-resolved |
| Ollama | `ollama` | None (local) | User-configured |
| OpenRouter | `openai-completions` | API key | User-configured |
| xAI (Grok) | `openai-completions` | API key | `grok-4-1-fast` |
| Mistral | `openai-completions` | API key | User-configured |
| MiniMax | `anthropic-messages` | API key | `MiniMax-VL-01` (vision) |
| Moonshot | `openai-completions` | API key | User-configured |
| vLLM | `openai-completions` | API key | Auto-discovered |

**Multi-provider features:**
- **Fallback chain:** Primary model + `fallbacks[]` tried in order on failure
- **Auth profile cooldown:** Rate-limited providers enter exponential backoff (5min → 25min → 60min cap)
- **Billing detection:** HTTP 402 / "insufficient credits" → separate `disabledUntil` with same backoff
- **Probe-on-cooldown:** Near cooldown expiry, probe attempts every 30s
- **Context overflow:** Not retried across models (smaller context would fare worse)

**Cost tracking:** Per-model `{ input, output, cacheRead, cacheWrite }` rates in USD/1M tokens. Token counts normalized across provider naming conventions.

---

### Embedding Providers

| Provider | Endpoint | Default Model | Max Tokens |
|---|---|---|---|
| OpenAI | `POST /v1/embeddings` | `text-embedding-3-small` | 8,192 |
| Google Gemini | `POST /v1beta/models/<m>:batchEmbedContents` | `gemini-embedding-001` | 2,048 |
| Voyage | `POST /v1/embeddings` | `voyage-4-large` | 32,000 |
| Mistral | `POST /v1/embeddings` | `mistral-embed` | — |
| Local | `node-llama-cpp` | `embeddinggemma-300m-qat-Q8_0.gguf` | — |

**Auto-selection order:** Local → OpenAI → Gemini → Voyage → Mistral. Falls back to FTS-only if no provider has keys.

---

### Text-to-Speech Providers

| Provider | Endpoint | Auth | Default Voice | Default Model |
|---|---|---|---|---|
| ElevenLabs | `POST /v1/text-to-speech/<voiceId>` | `xi-api-key` header | `pMsXgVXv3BLzUgSXRplE` | `eleven_multilingual_v2` |
| OpenAI | `POST /v1/audio/speech` | Bearer token | `alloy` | `gpt-4o-mini-tts` |
| Edge (Microsoft) | WebSocket (free, no key) | None | `en-US-MichelleNeural` | — |

**Auto-selection:** OpenAI key present → openai; ElevenLabs key → elevenlabs; else → edge.
**Long text:** Texts > 1500 chars are LLM-summarized before TTS.
**Output formats:** MP3 (default), Opus (Telegram), PCM (telephony).

---

### Web Search Providers

| Provider | Endpoint | Auth | Default Results |
|---|---|---|---|
| Brave | `GET /res/v1/web/search` | `X-Subscription-Token` | 5 |
| Perplexity | `POST /chat/completions` (direct or via OpenRouter) | Bearer token | — |
| Grok (xAI) | `POST /v1/responses` with `web_search` tool | Bearer token | — |

**Common features:** In-memory cache (5min TTL), freshness filters (`pd`/`pw`/`pm`/`py`), 30s timeout.

---

### Web Fetch

| Method | Endpoint | Fallback |
|---|---|---|
| Direct HTTP | Target URL via SSRF-safe fetch | Firecrawl |
| Firecrawl | `POST /v2/scrape` | — |

**Direct fetch:** Max 50K chars, 2MB response, 3 redirects, Readability extraction. Detects Cloudflare Markdown-for-Agents header.
**Firecrawl:** Tried on direct failure (network error, non-2xx, empty Readability). 48h cache, auto-proxy.

---

### Media Understanding

| Capability | Providers (priority order) | Default Limits |
|---|---|---|
| Image | Session model → imageModel config → Gemini CLI → OpenAI → Anthropic → Google → MiniMax → xAI | 10 MB, 60s, 500 char description |
| Audio | Session model → sherpa-onnx → whisper-cli → whisper → OpenAI → Groq → Deepgram → Google → Mistral | 20 MB, 60s |
| Video | Session model → Google | 50 MB, 120s, 500 char description |

---

### Browser Automation

| Component | Technology | Notes |
|---|---|---|
| Browser | Chrome/Chromium (CDP) | Launched with `--remote-debugging-port` |
| Automation lib | Playwright (`chromium.connectOverCDP`) | Attaches to running browser |
| Profiles | Multiple CDP ports | Separate browser contexts |
| SSRF guard | Hostname allowlist | Blocks private network by default |

---

### Networking & Discovery

| Integration | Protocol | Purpose |
|---|---|---|
| Tailscale | IP detection + serve/funnel APIs | Secure external access |
| Bonjour/mDNS | `_openclaw-gw._tcp` on `local` | LAN gateway discovery |
| APNS | Apple Push Notification Service | iOS push notifications |
| Google Pub/Sub | Webhook subscription | Gmail push notifications |

---

## 4.6 Credentials Summary

| Integration | Environment Variable | Config Key |
|---|---|---|
| Anthropic | `ANTHROPIC_API_KEY` | `models.providers.anthropic.apiKey` |
| OpenAI | `OPENAI_API_KEY` | `models.providers.openai.apiKey` |
| Google/Gemini | `GOOGLE_API_KEY` / `GEMINI_API_KEY` | `models.providers.google.apiKey` |
| AWS Bedrock | `AWS_PROFILE` / `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` | `models.bedrockDiscovery.*` |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN` / `GH_TOKEN` | Auth profiles store |
| xAI (Grok) | `XAI_API_KEY` | `tools.web.search.grok.apiKey` |
| OpenRouter | `OPENROUTER_API_KEY` | `models.providers.openrouter.apiKey` |
| Ollama | — (local, no key) | `models.providers.ollama.*` |
| ElevenLabs | `ELEVENLABS_API_KEY` / `XI_API_KEY` | `messages.tts.elevenlabs.apiKey` |
| Brave Search | `BRAVE_API_KEY` | `tools.web.search.apiKey` |
| Perplexity | `PERPLEXITY_API_KEY` | `tools.web.search.perplexity.apiKey` |
| Firecrawl | `FIRECRAWL_API_KEY` | `tools.web.fetch.firecrawl.apiKey` |
| Deepgram | — | `models.providers.deepgram.apiKey` |
| Voyage | `VOYAGE_API_KEY` | Auth profiles store |
| Mistral | `MISTRAL_API_KEY` | Auth profiles store |
| Gateway token | `OPENCLAW_GATEWAY_TOKEN` | `gateway.auth.token` |
| Hooks token | — | `hooks.token` |
