# 04 — API & Agent Capabilities

## 4.1 API Strategy

IronClaw exposes three HTTP surfaces:

| Surface | Audience | Auth | Base Path |
|---------|----------|------|-----------|
| **Web Gateway** | Browser UI, external clients | Bearer token (constant-time comparison) | `/api/...` |
| **Orchestrator** | Worker containers (internal) | Per-job bearer token | `/worker/{job_id}/...` |
| **OpenAI-Compatible** | Drop-in replacement for OpenAI clients | Bearer token (same as gateway) | `/v1/...` |

All protected routes require `Authorization: Bearer {token}` header or `?token={token}` query parameter. CORS is restricted to same-origin (localhost, 127.0.0.1). Max request body: 1 MB. Security headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`.

---

## 4.2 Web Gateway Endpoints

### Chat

| Method | Path | Handler | Rate Limit | Description |
|--------|------|---------|------------|-------------|
| `POST` | `/api/chat/send` | `chat_send_handler` | 30/60s | Send message to agent |
| `POST` | `/api/chat/approval` | `chat_approval_handler` | — | Approve/deny/always tool execution |
| `POST` | `/api/chat/auth-token` | `chat_auth_token_handler` | — | Submit extension auth token |
| `POST` | `/api/chat/auth-cancel` | `chat_auth_cancel_handler` | — | Cancel auth flow |
| `GET` | `/api/chat/events` | `chat_events_handler` | — | SSE event stream |
| `GET` | `/api/chat/ws` | `chat_ws_handler` | — | WebSocket upgrade |
| `GET` | `/api/chat/history` | `chat_history_handler` | — | Paginated conversation history |
| `GET` | `/api/chat/threads` | `chat_threads_handler` | — | List all threads |
| `POST` | `/api/chat/thread/new` | `chat_new_thread_handler` | — | Create new thread |

### Memory / Workspace

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/api/memory/tree` | `memory_tree_handler` | List workspace directory tree |
| `GET` | `/api/memory/list` | `memory_list_handler` | List workspace entries |
| `GET` | `/api/memory/read` | `memory_read_handler` | Read a workspace file by path |
| `POST` | `/api/memory/write` | `memory_write_handler` | Write/append to a workspace file |
| `POST` | `/api/memory/search` | `memory_search_handler` | Hybrid search (FTS + vector via RRF) |

### Jobs

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/api/jobs` | `jobs_list_handler` | List all jobs (filterable by status) |
| `GET` | `/api/jobs/summary` | `jobs_summary_handler` | Aggregate job statistics |
| `GET` | `/api/jobs/{id}` | `jobs_detail_handler` | Job details + action history |
| `POST` | `/api/jobs/{id}/cancel` | `jobs_cancel_handler` | Cancel a running/pending job |
| `POST` | `/api/jobs/{id}/restart` | `jobs_restart_handler` | Restart a failed/cancelled job |
| `POST` | `/api/jobs/{id}/prompt` | `jobs_prompt_handler` | Send follow-up prompt to Claude Code job |
| `GET` | `/api/jobs/{id}/events` | `jobs_events_handler` | SSE stream of job events |
| `GET` | `/api/jobs/{id}/files/list` | `job_files_list_handler` | List files in sandbox workspace |
| `GET` | `/api/jobs/{id}/files/read` | `job_files_read_handler` | Read file from sandbox workspace |

### Routines

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/api/routines` | `routines_list_handler` | List all routines |
| `GET` | `/api/routines/summary` | `routines_summary_handler` | Routine execution statistics |
| `GET` | `/api/routines/{id}` | `routines_detail_handler` | Routine config + state |
| `POST` | `/api/routines/{id}/trigger` | `routines_trigger_handler` | Manually fire a routine |
| `POST` | `/api/routines/{id}/toggle` | `routines_toggle_handler` | Enable/disable routine |
| `DELETE` | `/api/routines/{id}` | `routines_delete_handler` | Delete routine + run history |
| `GET` | `/api/routines/{id}/runs` | `routines_runs_handler` | Routine execution history |

### Skills

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/api/skills` | `skills_list_handler` | List loaded skills with trust levels |
| `POST` | `/api/skills/search` | `skills_search_handler` | Search ClawHub registry |
| `POST` | `/api/skills/install` | `skills_install_handler` | Install skill from registry |
| `DELETE` | `/api/skills/{name}` | `skills_remove_handler` | Remove installed skill |

### Extensions

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/api/extensions` | `extensions_list_handler` | List installed extensions |
| `GET` | `/api/extensions/tools` | `extensions_tools_handler` | List tools from extensions |
| `GET` | `/api/extensions/registry` | `extensions_registry_handler` | Browse extension catalog |
| `POST` | `/api/extensions/install` | `extensions_install_handler` | Install extension |
| `POST` | `/api/extensions/{name}/activate` | `extensions_activate_handler` | Activate extension |
| `POST` | `/api/extensions/{name}/remove` | `extensions_remove_handler` | Remove extension |
| `GET` | `/api/extensions/{name}/setup` | `extensions_setup_handler` | Get setup form |
| `POST` | `/api/extensions/{name}/setup` | `extensions_setup_submit_handler` | Submit setup form |

### Settings

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/api/settings` | `settings_list_handler` | List all settings |
| `GET` | `/api/settings/export` | `settings_export_handler` | Export as JSON |
| `POST` | `/api/settings/import` | `settings_import_handler` | Import from JSON |
| `GET` | `/api/settings/{key}` | `settings_get_handler` | Get single setting |
| `PUT` | `/api/settings/{key}` | `settings_set_handler` | Set single setting |
| `DELETE` | `/api/settings/{key}` | `settings_delete_handler` | Delete setting |

### Logs

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/api/logs/events` | `logs_events_handler` | SSE stream of application logs |
| `GET` | `/api/logs/level` | `logs_level_get_handler` | Get current log level |
| `PUT` | `/api/logs/level` | `logs_level_set_handler` | Set log level at runtime |

### Other

| Method | Path | Handler | Auth | Description |
|--------|------|---------|------|-------------|
| `GET` | `/api/health` | `health_handler` | **No** | Health check (public) |
| `GET` | `/api/gateway/status` | `gateway_status_handler` | Yes | Server uptime + stats |
| `GET` | `/api/pairing/{channel}` | `pairing_list_handler` | Yes | Pending device pairings |
| `POST` | `/api/pairing/{channel}/approve` | `pairing_approve_handler` | Yes | Approve device pairing |
| `POST` | `/v1/chat/completions` | `chat_completions_handler` | Yes | OpenAI-compatible chat API |
| `GET` | `/v1/models` | `models_handler` | Yes | OpenAI-compatible model list |

### Static Assets (no auth)

| Path | Content |
|------|---------|
| `/` | HTML (single-page app) |
| `/style.css` | CSS |
| `/app.js` | JavaScript |
| `/favicon.ico` | Favicon |
| `/projects/{id}/{*path}` | Project file serving (auth required) |

---

## 4.3 Orchestrator Internal API

The orchestrator runs on the host and serves workers running inside Docker containers. Workers authenticate with a per-job bearer token injected as `IRONCLAW_WORKER_TOKEN` env var.

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/worker/{job_id}/job` | `get_job` | Get job title + description + project_dir |
| `POST` | `/worker/{job_id}/llm/complete` | `llm_complete` | Proxy text completion through host LLM |
| `POST` | `/worker/{job_id}/llm/complete_with_tools` | `llm_complete_with_tools` | Proxy tool-calling completion through host LLM |
| `POST` | `/worker/{job_id}/status` | `report_status` | Report worker state + iteration count |
| `POST` | `/worker/{job_id}/complete` | `report_complete` | Report job completion (success/fail) |
| `POST` | `/worker/{job_id}/event` | `job_event_handler` | Post event (message/tool_use/tool_result/result) |
| `GET` | `/worker/{job_id}/prompt` | `get_prompt_handler` | Poll for follow-up prompts (204 if none) |
| `GET` | `/worker/{job_id}/credentials` | `get_credentials_handler` | Fetch credential grants (204 if none) |
| `GET` | `/health` | `health_check` | Health check (no auth) |

**Key data flow:** Workers never call LLM providers directly. All LLM requests are proxied through the orchestrator, which holds the API keys and applies rate limiting, caching, and circuit breaking.

---

## 4.4 Agent Tool Catalog

These are the tools the LLM agent can invoke during reasoning. Each tool is described as an agent-callable function.

### Memory Tools

```
memory_search(query: string, limit?: int) -> SearchResult[]
```
Hybrid search (FTS + vector via Reciprocal Rank Fusion) across workspace memory. Returns snippets with relevance scores. **Must be called before answering questions about prior work.**

```
memory_write(content: string, target?: string, append?: bool) -> WriteConfirmation
```
Write to persistent database-backed memory. Targets: `"memory"` (MEMORY.md), `"daily_log"` (today's log), `"heartbeat"` (HEARTBEAT.md), or custom path. Rate-limited: 20/min, 200/hour.

```
memory_read(path: string) -> FileContent
```
Read a workspace memory file by path (e.g., `"MEMORY.md"`, `"daily/2024-01-15.md"`).

```
memory_tree(path?: string, depth?: int) -> TreeListing
```
View workspace directory structure. Default depth: 1. Max depth: 10.

### Job Tools

```
create_job(title: string, description: string, wait?: bool, mode?: "worker"|"claude_code",
           project_dir?: string, credentials?: {secret_name: env_var}) -> JobResult
```
Create and execute a sandboxed job. Mode `"worker"` uses IronClaw sub-agent; `"claude_code"` uses Claude CLI. Set `wait=false` for async execution. Credentials are injected as env vars in the container. Rate-limited: 5/min, 30/hour. Timeout: 660s.

```
list_jobs(filter?: "active"|"completed"|"failed"|"all") -> JobSummary[]
```
List jobs with IDs, titles, and status.

```
job_status(job_id: string) -> JobDetail
```
Detailed status of a specific job. Accepts full UUID or short prefix.

```
cancel_job(job_id: string) -> Confirmation
```
Cancel a running or pending job. Requires approval (unless auto-approved).

```
job_events(job_id: string, limit?: int) -> EventLog[]
```
Read event log from sandbox job (messages, tool calls, results, status changes). Output is sanitized.

```
job_prompt(job_id: string, content: string, done?: bool) -> Confirmation
```
Send follow-up prompt to a running Claude Code job. Set `done=true` to signal wrap-up. Requires approval.

### Routine Tools

```
routine_create(name: string, trigger_type: "cron"|"event"|"webhook"|"manual",
               prompt: string, schedule?: string, event_pattern?: string,
               event_channel?: string, context_paths?: string[],
               action_type?: "lightweight"|"full_job",
               cooldown_secs?: int) -> RoutineDetail
```
Create a scheduled or reactive routine. Cron uses 6-field format. Event uses regex pattern matching.

```
routine_list() -> RoutineSummary[]
```
List all routines with status, trigger info, and next fire time.

```
routine_update(name: string, enabled?: bool, prompt?: string,
               schedule?: string, description?: string) -> RoutineDetail
```
Update routine configuration. Only specified fields change.

```
routine_delete(name: string) -> Confirmation
```
Permanently delete a routine and its run history.

```
routine_history(name: string, limit?: int) -> RoutineRun[]
```
View execution history with status, duration, and results.

### File Tools (Sandbox)

```
read_file(path: string, offset?: int, limit?: int) -> FileContent
```
Read from the local filesystem (NOT workspace memory). Max 1 MB. Requires approval. Domain: Container.

```
write_file(path: string, content: string) -> Confirmation
```
Write to the local filesystem. Max 5 MB. Auto-creates parent directories. Requires approval. Domain: Container.

```
list_dir(path: string) -> DirectoryListing
```
List directory contents. Domain: Container.

```
apply_patch(path: string, patch: string) -> PatchResult
```
Apply a targeted edit patch to a file. Domain: Container.

### Shell Tool (Sandbox)

```
shell(command: string, timeout_secs?: int) -> ShellOutput
```
Execute a shell command in the sandbox. Max output: 64 KB. Default timeout: 120s. Domain: Container.

**Security layers:**
- Blocked commands: `rm -rf /`, fork bombs, `mkfs`, etc.
- Dangerous patterns: `sudo`, `eval`, `$(curl`, pipe to shell
- Injection detection: base64-to-shell, hex escapes, DNS exfiltration, netcat
- Environment scrubbing: only safe env vars forwarded (PATH, HOME, LANG, EDITOR, etc.)
- Some patterns require explicit approval even with auto-approve

### HTTP Tool

```
http(method: "GET"|"POST"|"PUT"|"DELETE"|"PATCH", url: string,
     headers?: {name, value}[], body?: any, timeout_secs?: int) -> HttpResponse
```
Make HTTP requests. HTTPS only, no localhost/private IPs, max response 5 MB. Credentials are automatically injected for configured secrets via the proxy. Output is sanitized.

### Extension Tools

```
tool_search(query: string, discover?: bool) -> ExtensionEntry[]
```
Search for extensions (channels, tools, MCP servers). Set `discover=true` for online search (slower, 5-15s).

```
tool_install(name: string, url?: string, kind?: "mcp_server"|"wasm_tool"|"wasm_channel") -> InstallResult
```
Install an extension. Auto-detects type if not specified. Requires approval.

```
tool_auth(name: string) -> AuthInstructions
```
Initiate auth for an extension. Returns OAuth URL or manual token instructions.

```
tool_activate(name: string) -> Confirmation
```
Activate an installed extension.

```
tool_remove(name: string) -> Confirmation
```
Remove an installed extension.

### Skill Tools

```
skill_list(verbose?: bool) -> SkillEntry[]
```
List all loaded skills with trust level, source, and activation keywords.

```
skill_search(query: string) -> SkillEntry[]
```
Search ClawHub registry and local skills.

```
skill_install(name: string, url?: string, content?: string) -> InstallResult
```
Install a skill from registry, URL, or raw SKILL.md content. Requires approval.

```
skill_remove(name: string) -> Confirmation
```
Remove an installed skill.

### Utility Tools

```
echo(message: string) -> string
```
Echo back input (testing/confirmation).

```
time() -> string
```
Current UTC timestamp.

```
json(action: "parse"|"format"|"query", data: any, query?: string) -> any
```
JSON manipulation utility.

---

## 4.5 Tool Trust & Approval Matrix

| Tool | Approval | Available to Installed Skills | Sanitized Output |
|------|----------|------------------------------|------------------|
| `memory_search` | Never | Yes | No |
| `memory_write` | Never | **No** | No |
| `memory_read` | Never | Yes | No |
| `memory_tree` | Never | Yes | No |
| `create_job` | Never | No | No |
| `list_jobs` | Never | No | No |
| `job_status` | Never | No | No |
| `cancel_job` | UnlessAutoApproved | No | No |
| `job_events` | Never | No | Yes |
| `job_prompt` | UnlessAutoApproved | No | No |
| `routine_*` | Never | No | No |
| `read_file` | UnlessAutoApproved | No | Yes |
| `write_file` | UnlessAutoApproved | No | No |
| `shell` | Varies by pattern | No | Yes |
| `http` | Never | No | Yes |
| `tool_install` | UnlessAutoApproved | No | No |
| `tool_search` | Never | No | No |
| `skill_install` | UnlessAutoApproved | No | No |
| `skill_list` | Never | Yes | No |
| `skill_search` | Never | Yes | No |
| `echo` | Never | Yes | No |
| `time` | Never | Yes | No |
| `json` | Never | Yes | No |

---

## 4.6 LLM Provider Interface

### Trait: `LlmProvider`

All LLM backends implement this trait:

```rust
trait LlmProvider: Send + Sync {
    fn model_name(&self) -> &str;
    fn cost_per_token(&self) -> (Decimal, Decimal);           // (input, output)
    async fn complete(&self, req: CompletionRequest) -> Result<CompletionResponse, LlmError>;
    async fn complete_with_tools(&self, req: ToolCompletionRequest)
        -> Result<ToolCompletionResponse, LlmError>;
    async fn list_models(&self) -> Result<Vec<String>, LlmError>;
    async fn model_metadata(&self) -> Result<ModelMetadata, LlmError>;
    fn effective_model_name(&self, requested: Option<&str>) -> String;
    fn active_model_name(&self) -> String;
    fn set_model(&self, model: &str) -> Result<(), LlmError>;
    fn calculate_cost(&self, input_tokens: u32, output_tokens: u32) -> Decimal;
}
```

### Core Message Types

```rust
struct ChatMessage {
    role: Role,                           // System | User | Assistant | Tool
    content: String,
    tool_call_id: Option<String>,         // Links tool result to its call
    name: Option<String>,                 // Tool name for role=Tool
    tool_calls: Option<Vec<ToolCall>>,    // Present when role=Assistant requests tools
}

struct ToolCall {
    id: String,                           // Unique call ID
    name: String,                         // Tool name
    arguments: serde_json::Value,         // JSON parameters
}

struct ToolDefinition {
    name: String,
    description: String,
    parameters: serde_json::Value,        // JSON Schema
}

enum FinishReason { Stop, Length, ToolUse, ContentFilter, Unknown }
```

### Request/Response Flow

```
CompletionRequest                    CompletionResponse
┌──────────────────────┐             ┌──────────────────────┐
│ messages: [ChatMsg]  │────────────►│ content: String      │
│ model: Option<str>   │   complete  │ input_tokens: u32    │
│ max_tokens: Option   │             │ output_tokens: u32   │
│ temperature: Option  │             │ finish_reason: enum  │
│ stop_sequences: []   │             └──────────────────────┘
│ metadata: HashMap    │
└──────────────────────┘

ToolCompletionRequest                ToolCompletionResponse
┌──────────────────────┐             ┌──────────────────────┐
│ messages: [ChatMsg]  │  complete   │ content: Option<str> │
│ tools: [ToolDef]     │──────with──►│ tool_calls: [Call]   │
│ model: Option<str>   │   tools     │ input_tokens: u32    │
│ max_tokens: Option   │             │ output_tokens: u32   │
│ temperature: Option  │             │ finish_reason: enum  │
│ tool_choice: Option  │             └──────────────────────┘
│ metadata: HashMap    │
└──────────────────────┘
```

---

## 4.7 External Integrations

### LLM Providers

| Provider | Auth | Base URL | Key Features |
|----------|------|----------|-------------|
| **NEAR AI** (default) | Session token (`sess_xxx`) via browser OAuth, or API key | `https://private.near.ai` (session) / `https://cloud-api.near.ai` (API key) | Dual auth, tool messages flattened to text |
| **OpenAI** | API key (`sk-xxx`) | Configurable | Full tool calling support |
| **Anthropic** | API key via `x-api-key` header | Configurable | Claude models |
| **Ollama** | None | `http://localhost:11434` | Local models, no auth |
| **OpenAI-Compatible** | Optional API key | Configurable `LLM_BASE_URL` | vLLM, LiteLLM, OpenRouter. Custom headers via `LLM_EXTRA_HEADERS` |
| **Tinfoil** | API key | `https://inference.tinfoil.sh/v1` | Hardware-attested TEE, private inference |

### Embedding Providers

| Provider | Models | Dimensions | Purpose |
|----------|--------|-----------|---------|
| **OpenAI** | `text-embedding-3-small` (default), `text-embedding-3-large`, `text-embedding-ada-002` | 1536 / 3072 / 1536 | Workspace semantic search |
| **NEAR AI** | Same models via NEAR AI API | Same | Alternative provider |

### ClawHub Registry

| Operation | Endpoint | Notes |
|-----------|----------|-------|
| Search skills | `GET /api/v1/search?q={query}` | Cache TTL: 5 min, max results: 25 |
| Download skill | Via catalog response URL | Installs to `~/.ironclaw/installed_skills/` |

### MCP Servers

| Transport | Status | Protocol |
|-----------|--------|----------|
| HTTP | Implemented | JSON-RPC 2.0 |
| stdio | Not implemented | — |

### Docker Engine

| Operation | Method | Purpose |
|-----------|--------|---------|
| Create container | Docker API | Sandbox job execution |
| Start/stop container | Docker API | Lifecycle management |
| List containers | Docker API | Job tracking |
| Remove container | Docker API | Cleanup |

### Network Proxy Credential Injection

| Target Domain | Auth Type | Header |
|---------------|-----------|--------|
| `api.openai.com` | Bearer | `Authorization: Bearer {OPENAI_API_KEY}` |
| `api.anthropic.com` | Custom header | `x-api-key: {ANTHROPIC_API_KEY}` |
| `api.near.ai` | Bearer | `Authorization: Bearer {NEARAI_API_KEY}` |
| Custom | Via `CredentialMapping` trait | Configurable per-domain |

---

## 4.8 Real-Time Event Types

Events broadcast over SSE and WebSocket:

| Event | Payload | When |
|-------|---------|------|
| `Response` | `{ content, thread_id }` | Agent sends final text response |
| `Thinking` | `{ message, thread_id? }` | Agent reasoning status |
| `StreamChunk` | `{ content, thread_id? }` | Incremental response streaming |
| `ToolStarted` | `{ name, thread_id? }` | Tool execution begins |
| `ToolCompleted` | `{ name, success, thread_id? }` | Tool execution ends |
| `ToolResult` | `{ name, preview, thread_id? }` | Tool output preview |
| `ApprovalNeeded` | `{ request_id, tool_name, description, parameters, thread_id? }` | Tool requires user approval |
| `AuthRequired` | `{ extension_name, instructions?, auth_url?, setup_url? }` | Extension needs authentication |
| `AuthCompleted` | `{ extension_name, success, message }` | Auth flow completed |
| `Status` | `{ message, thread_id? }` | General status update |
| `Error` | `{ message, thread_id? }` | Error notification |
| `Heartbeat` | (empty) | SSE keepalive (30s interval) |
| `JobStarted` | `{ job_id, title, browse_url }` | Sandbox job created |
| `JobMessage` | `{ job_id, role, content }` | Message from sandbox worker |
| `JobToolUse` | `{ job_id, tool_name, input }` | Worker invoked a tool |
| `JobToolResult` | `{ job_id, tool_name, output }` | Worker tool result |
| `JobStatus` | `{ job_id, message }` | Worker status change |
| `JobResult` | `{ job_id, status, session_id? }` | Job completed/failed |

### WebSocket Client → Server Messages

| Type | Payload | Purpose |
|------|---------|---------|
| `Message` | `{ content, thread_id? }` | Send chat message |
| `Approval` | `{ request_id, action: "approve"/"always"/"deny", thread_id? }` | Respond to approval request |
| `AuthToken` | `{ extension_name, token }` | Submit extension auth token |
| `AuthCancel` | (empty) | Cancel auth flow |
| `Ping` | (empty) | Keepalive (responds with `Pong`) |
