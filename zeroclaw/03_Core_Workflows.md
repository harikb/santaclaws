# 03 — Core Workflows

> ZeroClaw Technical Blueprint | File 3 of 5

---

## 1. Agent Orchestration Loop

The primary workflow — every user message flows through this loop regardless of entry point (CLI, channel, webhook, cron).

### 1.1 Sequence Diagram

```
┌──────┐     ┌─────────┐     ┌────────┐     ┌──────────┐     ┌───────┐     ┌────────┐
│ User │     │ Channel  │     │ Hooks  │     │  Agent   │     │Provider│    │ Tools  │
└──┬───┘     └────┬─────┘     └───┬────┘     └────┬─────┘     └───┬───┘     └───┬────┘
   │  message     │               │                │               │             │
   │─────────────►│               │                │               │             │
   │              │ on_message_   │                │               │             │
   │              │ received()    │                │               │             │
   │              │──────────────►│                │               │             │
   │              │  Continue(msg)│                │               │             │
   │              │◄──────────────│                │               │             │
   │              │               │  process_msg() │               │             │
   │              │──────────────────────────────►│               │             │
   │              │               │                │ recall()      │             │
   │              │               │                │──►Memory      │             │
   │              │               │                │◄── entries    │             │
   │              │               │                │               │             │
   │              │               │                │ build_prompt() │             │
   │              │               │                │───────┐       │             │
   │              │               │                │◄──────┘       │             │
   │              │               │                │               │             │
   │              │               │  before_llm_   │               │             │
   │              │               │  call()        │               │             │
   │              │               │◄───────────────│               │             │
   │              │               │  Continue(msgs)│               │             │
   │              │               │───────────────►│               │             │
   │              │               │                │  chat()       │             │
   │              │               │                │──────────────►│             │
   │              │               │                │  ChatResponse │             │
   │              │               │                │◄──────────────│             │
   │              │               │                │               │             │
   │              │               │                │──[if tool_calls]────────────►│
   │              │               │                │               │  execute()  │
   │              │               │                │◄──────────────────ToolResult│
   │              │               │                │               │             │
   │              │               │                │──[loop up to max_iterations]│
   │              │               │                │               │             │
   │              │               │  on_message_   │               │             │
   │              │               │  sending()     │               │             │
   │              │               │◄───────────────│               │             │
   │              │  send()       │                │               │             │
   │              │◄──────────────────────────────│               │             │
   │  response    │               │                │               │             │
   │◄─────────────│               │                │               │             │
```

### 1.2 Step-by-Step Flow

#### Step 1: Message Reception

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| User | Sends message via CLI, Telegram, Discord, Slack, webhook, etc. | Channel listener receives `ChannelMessage { id, sender, reply_target, content, channel, timestamp }` | — |
| System | Run `hooks.run_on_message_received(message)` | If `Cancel(reason)`: message dropped, no further processing. If `Continue(msg)`: proceed with (potentially modified) message. | Hook audit log |

#### Step 2: Runtime Command Check

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Check if content starts with `/models` or `/model <name>` | `/models`: return provider model list. `/model <name>`: switch session model. Short-circuit, no agent invocation. | Session model preference |

#### Step 3: Memory Context Loading

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Call `memory.recall(user_message, limit=5, session_id=None)` | Returns `Vec<MemoryEntry>` ranked by hybrid score (vector_weight=0.7, keyword_weight=0.3) | — |
| System | Filter entries: remove auto-saved assistant responses, remove entries below `min_relevance_score` (0.4) | Filtered list of relevant memory entries | — |
| System | Format as `[Memory context]\n- key: content\n...` | Prepended to user message | — |

#### Step 4: System Prompt Construction

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Build system prompt from 7 sections | Concatenated prompt string | — |

**Prompt sections (in order):**

| # | Section | Content |
|---|---|---|
| 1 | Identity | AIEOS identity document OR workspace files: `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`, `MEMORY.md` (each truncated at 20,000 chars) |
| 2 | Tools | Tool name, description, JSON Schema parameters for each registered tool. Includes dispatcher instructions (XML tag format or native mode). |
| 3 | Safety | Hardcoded rules: no data exfiltration, prefer `trash` over `rm`, ask before destructive ops, no credentials in logs |
| 4 | Skills | Available open skills in `<available_skills>` XML block (full or compact mode) |
| 5 | Workspace | `Working directory: {path}` |
| 6 | DateTime | `{YYYY-MM-DD HH:MM:SS} ({timezone})` |
| 7 | Runtime | `Host: {hostname} | OS: {os_type} | Model: {model}` |

#### Step 5: Query Classification & Model Routing

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Evaluate `QueryClassificationConfig.rules` sorted by descending priority | For each rule: check `min_length`, `max_length`, keyword match (case-insensitive), pattern match (case-sensitive) | — |
| System | If rule matches and hint exists in `model_routes` | Override effective model to the routed provider/model for this request | — |
| System | If no match | Use default provider + model from config | — |

#### Step 6: Conversation History Assembly

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Load per-sender history (max 50 messages) | `Vec<ChatMessage>` from in-memory store | — |
| System | Normalize consecutive same-role messages | Merge to prevent provider rejection | History (in-memory) |
| System | Append enriched user message | Push `{ role: "user", content: memory_context + user_text }` | History (in-memory) |

#### Step 7: Tool Iteration Loop

This loop runs up to `max_tool_iterations` times (default: 10).

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Check cancellation token | If cancelled (new message from same sender on interruptible channel): `bail!("cancelled")` | — |
| System | Prepare multimodal content | Resolve `[image:...]` markers to base64 payloads if provider supports vision | — |
| System | Run `hooks.run_before_llm_call(messages, model)` | May modify messages or model; `Cancel` aborts entire turn | — |
| System | Call `provider.chat(messages, model, temperature, tool_specs)` | Returns `ChatResponse { text, tool_calls, usage, reasoning_content }` | — |
| System | Record observability event | `ObserverEvent::LlmResponse { provider, model, duration, success, tokens }` | Observer metrics |
| System | Record cost | `CostRecord { model, input_tokens, output_tokens, cost_usd }` | Cost tracker |

**If no tool calls in response:**

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Push assistant message to history | `{ role: "assistant", content: response_text }` | History |
| System | Stream response in chunks (≥80 chars) via `on_delta` callback | Draft updates on channels that support it | — |
| System | Return final response text | Loop exits | — |

**If tool calls present:**

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Parse tool calls (7 format fallbacks) | Structured: OpenAI-format `tool_calls` array. Text: XML, JSON, markdown code block, GLM-style | — |
| System | Normalize tool name aliases | `bash`→`shell`, `fileread`→`file_read`, `filewrite`→`file_write`, etc. | — |
| System | Push `AssistantToolCalls` to history | `{ text, tool_calls: [{id, name, arguments}], reasoning_content }` | History |
| System | For each tool call: run `hooks.run_before_tool_call(name, args)` | `Cancel`: skip tool, inject error result. `Continue`: proceed (may modify name/args). | — |
| System | For each tool call: check approval | See Step 7a below | — |
| System | For each tool call: check duplicate signature | Skip if `(name, canonical_args)` already executed this turn | — |
| System | Execute tools (parallel or serial based on config and approval needs) | `Vec<ToolResult { success, output, error }>` | — |
| System | Scrub credentials from output | Regex replaces `token=`, `api_key=`, etc. with `[REDACTED]` | — |
| System | Run `hooks.fire_after_tool_call(name, result, duration)` per tool | Void, fire-and-forget, parallel | — |
| System | Push `ToolResults` to history | Native: `{ role: "tool", tool_call_id, content }` per result. XML: `<tool_result>` block. | History |
| System | Check history size; compact if > `max_history_messages` | LLM-assisted summarization of oldest messages into bullet points (fallback: truncation) | History |
| System | Loop back to LLM call | Next iteration | — |

**If max iterations exhausted:**

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Bail with error | `"Agent exceeded maximum tool iterations (10)."` | — |

#### Step 7a: Approval Sub-Flow (Supervised Mode, CLI Only)

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Check `approval.needs_approval(tool_name)` | Decision tree: Full→no, ReadOnly→no (blocked by policy), always_ask→yes, auto_approve→no, session_allowlist→no, Supervised default→yes | — |
| System | If approval needed and channel is CLI | Print to stderr: `[Approval Required]\nTool: {name}\nArgs: {summary}` | — |
| CLI Operator | Types `y`/`n`/`a` | `Yes`: proceed. `No`: skip tool, inject error. `Always`: add to session allowlist, proceed. | Session allowlist, audit log |
| System | If approval needed and channel is NOT CLI | Auto-approve (interactive approval impossible in async channels) | — |

#### Step 8: Response Delivery

| Actor | Action | System Response | Data Updated |
|---|---|---|---|
| System | Run `hooks.run_on_message_sending(channel, recipient, content)` | May modify or `Cancel` (suppress reply) | — |
| System | Cap response at 20,000 chars | Truncate if needed | — |
| System | Finalize draft (if draft streaming active) or send new message | `channel.send(SendMessage { content, recipient, thread_ts })` | — |
| System | Auto-save assistant response to memory (if enabled) | `memory.store(session_key, response)` | Memory DB |
| System | Append assistant message to sender history | `{ role: "assistant", content }` | History (in-memory) |
| System | Remove typing indicator, update reactions | Remove 👀, add ✅ (success) or ⚠️ (error/timeout) | — |

#### Error Paths

| Error Condition | Handling |
|---|---|
| Hook cancels message | Message dropped silently, logged |
| Provider API failure | Error propagated, ⚠️ reaction, sanitized error message sent to user |
| Tool execution failure | `ToolResult { success: false, error: Some(msg) }` fed back to LLM for self-correction |
| Context window overflow | History compacted (keep last 12 turns, truncate to 600 chars each), warning sent to user |
| Timeout (budget = `timeout_secs * min(iterations, 4)`) | `[Task timed out]` appended to history, timeout error sent to user |
| Cancellation (new message interrupts) | Current turn aborted, no response sent |
| Max iterations exceeded | Error message returned as final response |

---

## 2. Gateway Webhook Flow

The HTTP/WebSocket API entry point for remote clients and integrations.

### 2.1 Pairing Flow (Device Registration)

```
┌────────┐                        ┌─────────┐
│ Device │                        │ Gateway │
└───┬────┘                        └────┬────┘
    │                                  │
    │  POST /pair                      │
    │  X-Pairing-Code: {code}          │
    │─────────────────────────────────►│
    │                                  │── Rate limit check (10/min per IP)
    │                                  │── Extract X-Pairing-Code header
    │                                  │── PairingGuard.try_pair(code)
    │                                  │     ├─ Valid code → generate bearer token
    │                                  │     ├─ Invalid code → increment failure counter
    │                                  │     └─ Too many failures → lockout period
    │                                  │── Persist token to config.toml
    │  {"token": "...",                │
    │   "expires_in": ...}             │
    │◄─────────────────────────────────│
```

| Step | Actor | Action | Response | Data Updated |
|---|---|---|---|---|
| 1 | Device | `POST /pair` with `X-Pairing-Code` header | — | — |
| 2 | Gateway | Rate limit check: `pair_limiter.check(client_ip)` | 429 if exceeded | Rate limiter state |
| 3 | Gateway | Extract pairing code from header | 400 if missing | — |
| 4 | Gateway | `pairing.try_pair(code, rate_key)` | 401 if invalid code; 429 with `Retry-After` if locked out | Pairing state |
| 5 | Gateway | Persist new token to `config.toml` | — | Config file |
| 6 | Gateway | Return `{ token, expires_in }` | 200 OK | — |

### 2.2 Webhook Message Flow

```
┌────────┐                        ┌─────────┐                ┌───────┐
│ Client │                        │ Gateway │                │ Agent │
└───┬────┘                        └────┬────┘                └───┬───┘
    │  POST /webhook                   │                         │
    │  Authorization: Bearer {token}   │                         │
    │  {"message": "..."}              │                         │
    │─────────────────────────────────►│                         │
    │                                  │── Rate limit (60/min)   │
    │                                  │── Bearer token auth     │
    │                                  │── Webhook HMAC verify   │
    │                                  │── Idempotency check     │
    │                                  │── Body parse (64KB max) │
    │                                  │                         │
    │                                  │── process_message() ───►│
    │                                  │                         │── [full agent loop]
    │                                  │◄── response text ───────│
    │                                  │                         │
    │  {"response": "...",             │                         │
    │   "model": "..."}               │                         │
    │◄─────────────────────────────────│                         │
```

| Step | Actor | Action | Response | Data Updated |
|---|---|---|---|---|
| 1 | Client | `POST /webhook` with bearer token and JSON body | — | — |
| 2 | Gateway | Rate limit: `webhook_limiter.check(client_ip)` | 429 if exceeded | Rate limiter |
| 3 | Gateway | Auth: extract `Bearer` token, `pairing.is_authenticated(token)` (constant-time) | 401 if invalid | — |
| 4 | Gateway | HMAC: if webhook secret configured, verify `X-Webhook-Signature` SHA-256 | 403 if mismatch | — |
| 5 | Gateway | Idempotency: if `X-Idempotency-Key` present, `store.check_and_insert(key)` | 200 with cached response if duplicate | Idempotency store |
| 6 | Gateway | Parse body: `{ "message": String }` | 400 if malformed | — |
| 7 | Gateway | Auto-save user message to memory (if enabled) | — | Memory DB |
| 8 | Gateway | Run agent loop (simple mode: system prompt + user message, no tools) | — | Cost tracker, observer |
| 9 | Gateway | Return `{ response, model }` | 200 OK | Idempotency store (cache response) |
| 10 | Gateway | On error: sanitize (strip paths, credentials), return 500 | — | — |

### 2.3 WhatsApp Webhook Flow

| Step | Actor | Action | Response | Data Updated |
|---|---|---|---|---|
| 1 | Meta | `GET /whatsapp?hub.challenge=...` | Return `hub.challenge` value (verification) | — |
| 2 | Meta | `POST /whatsapp` with `X-Hub-Signature-256` | — | — |
| 3 | Gateway | Compute HMAC-SHA256 over raw body, constant-time compare | 403 if mismatch | — |
| 4 | Gateway | Parse WhatsApp payload, extract messages | — | — |
| 5 | Gateway | Per message: run full agent loop WITH tools | — | Memory, cost, observer |
| 6 | Gateway | Send reply via WhatsApp channel `send()` | — | — |

### 2.4 WebSocket Chat Flow

| Step | Actor | Action | Response | Data Updated |
|---|---|---|---|---|
| 1 | Client | `GET /ws/chat` → WebSocket upgrade | Connection established | — |
| 2 | Client | Send JSON frame: `{ "token": "...", "message": "..." }` | — | — |
| 3 | Gateway | Authenticate bearer token from first frame | Close connection if invalid | — |
| 4 | Gateway | Run agent loop, stream `on_delta` chunks as WS text frames | Incremental response frames | — |
| 5 | Gateway | Send final frame with `{ "done": true, "response": "..." }` | — | — |

### 2.5 SSE Event Stream

| Step | Actor | Action | Response | Data Updated |
|---|---|---|---|---|
| 1 | Client | `GET /api/events` with bearer auth | SSE stream opened | — |
| 2 | Gateway | Subscribe to `broadcast::channel` (256 event buffer) | — | — |
| 3 | System | Events emitted during agent processing | `data: { event JSON }\n\n` pushed to all subscribers | — |
| 4 | Client | Connection drops or server closes | Stream ended | — |

---

## 3. Cron / Daemon Autonomous Execution

### 3.1 Daemon Startup

```
┌──────────┐
│  main()  │
│ "daemon" │
└────┬─────┘
     │
     ▼
┌────────────────────────────────────────────┐
│           Daemon Supervisor                │
│                                            │
│  spawn_component_supervisor() for each:    │
│                                            │
│  ┌──────────────┐  ┌───────────────────┐   │
│  │ state_writer │  │     gateway       │   │
│  │ (5s flush)   │  │ (HTTP/WS server)  │   │
│  └──────────────┘  └───────────────────┘   │
│                                            │
│  ┌──────────────┐  ┌───────────────────┐   │
│  │   channels   │  │    heartbeat      │   │
│  │ (if config'd)│  │ (5+ min interval) │   │
│  └──────────────┘  └───────────────────┘   │
│                                            │
│  ┌──────────────┐                          │
│  │  scheduler   │                          │
│  │ (if enabled) │                          │
│  └──────────────┘                          │
│                                            │
│  Ctrl+C → abort all handles                │
└────────────────────────────────────────────┘
```

**Supervisor behavior per component:**
- Each component runs in its own `tokio::spawn` task.
- On exit (success or error): mark health error, bump restart counter, wait backoff, restart.
- Backoff: starts at `channel_initial_backoff_secs` (min 1s), doubles on failure up to `channel_max_backoff_secs`. Resets to initial on clean exit.
- State writer flushes `daemon_state.json` every 5 seconds.

### 3.2 Cron Scheduler Loop

```
┌───────────────┐
│  Scheduler    │
│  run() loop   │
└──────┬────────┘
       │
       ▼
┌──────────────────────────┐
│  interval.tick()         │◄─────────────────────────┐
│  (poll every 5+ seconds) │                          │
└──────────┬───────────────┘                          │
           │                                          │
           ▼                                          │
┌──────────────────────────┐                          │
│  query_due_jobs()        │                          │
│  WHERE next_run <= now   │                          │
│  AND enabled = 1         │                          │
└──────────┬───────────────┘                          │
           │                                          │
           ▼                                          │
┌──────────────────────────┐                          │
│  process_due_jobs()      │                          │
│  (up to max_concurrent   │                          │
│   in parallel)           │                          │
└──────────┬───────────────┘                          │
           │                                          │
           ▼                                          │
    ┌──────┴──────┐                                   │
    ▼             ▼                                   │
┌────────┐  ┌────────┐                               │
│ Shell  │  │ Agent  │                                │
│  Job   │  │  Job   │                                │
└───┬────┘  └───┬────┘                                │
    │           │                                     │
    ▼           ▼                                     │
┌──────────────────────────┐                          │
│  persist_job_result()    │                          │
│  - record CronRun        │                          │
│  - deliver if configured │                          │
│  - reschedule or delete  │                          │
└──────────┬───────────────┘                          │
           │                                          │
           └──────────────────────────────────────────┘
```

### 3.3 Cron Job Execution Detail

| Step | Actor | Action | System Response | Data Updated |
|---|---|---|---|---|
| 1 | Scheduler | Poll: `SELECT * FROM cron_jobs WHERE next_run <= now AND enabled = 1` | List of due jobs | — |
| 2 | Scheduler | For each job (up to `max_concurrent` in parallel): | — | — |
| 3a | Scheduler | **Shell job**: Security gates → `sh -c "{command}"` with 120s timeout | `JobOutput { stdout, stderr, exit_code, duration }` | Action tracker |
| 3b | Scheduler | **Agent job**: Security gates → `agent::run(config, prompt, tools, silent=true)` | Agent response text | Action tracker, cost, memory |
| 4 | Scheduler | On failure: retry up to `retries` times with exponential backoff (500ms→30s) + jitter | Non-retryable if "blocked by security policy" | — |
| 5 | Scheduler | Record run: `INSERT INTO cron_runs (job_id, started_at, finished_at, status, output, duration_ms)` | — | `cron_runs` table |
| 6 | Scheduler | Prune old runs: keep last `max_run_history` (50) per job | — | `cron_runs` table |
| 7 | Scheduler | Deliver output (if `delivery.mode == "announce"`): send to configured channel | — | Channel output |
| 8a | Scheduler | **One-shot job** (`delete_after_run && schedule.is_at_type()`): delete on success, disable on failure | — | `cron_jobs` table |
| 8b | Scheduler | **Recurring job**: compute next trigger, update `next_run`, `last_run`, `last_status`, `last_output` | — | `cron_jobs` table |

### 3.4 Security Gates for Cron Execution

Every cron job passes through these gates before execution:

```
1. security.can_act()
   └─ ReadOnly mode? → bail!("blocked: ReadOnly")

2. security.is_rate_limited()
   └─ Actions this hour >= max_actions_per_hour? → bail!("rate limit exceeded")

3. security.is_command_allowed(command)  [shell jobs only]
   └─ Not in allowlist? → bail!("command not allowed")
   └─ Contains subshell/redirect? → bail!("blocked")
   └─ Dangerous arg pattern? → bail!("blocked")

4. security.forbidden_path_argument(command)
   └─ Command references forbidden path? → bail!("forbidden path")

5. security.record_action()
   └─ Increment sliding window counter

6. Execute command
```

---

## 4. Channel Message Processing Flow

Detailed flow for messages arriving via persistent channels (Telegram, Discord, Slack, etc.).

### 4.1 Channel Startup & Listening

| Step | Actor | Action | System Response | Data Updated |
|---|---|---|---|---|
| 1 | Daemon | Call `start_channels()` | Build runtime context (provider, memory, tools, observer, hooks) | — |
| 2 | System | For each enabled channel in config | Instantiate channel, spawn supervised listener task | — |
| 3 | Channel | Call `channel.listen(message_tx)` | Long-poll, WebSocket, or webhook listener starts. Messages sent to shared `mpsc` channel. | — |
| 4 | System | Start `run_message_dispatch_loop(ctx, message_rx)` | Semaphore-gated (8–64 concurrent) message processing | — |

### 4.2 Per-Message Processing

| Step | Actor | Action | System Response | Data Updated |
|---|---|---|---|---|
| 1 | Dispatch loop | Acquire semaphore permit (backpressure) | Blocks if max concurrent reached | — |
| 2 | System | Build scope key: `{channel}_{sender}_{reply_target}` | — | — |
| 3 | System | Check for in-flight message from same scope | If interruptible: cancel previous via `CancellationToken`, wait for completion | In-flight map |
| 4 | System | Spawn `process_channel_message()` task | — | — |
| 5 | System | Add 👀 reaction to original message | — | — |
| 6 | System | Start typing indicator (every 4s) | `channel.send_typing(chat_id)` | — |
| 7 | System | Check config file modification time | If changed: hot-reload config from disk | Config |
| 8 | System | Build memory context, system prompt, load history | See Workflow 1 steps 3–6 | — |
| 9 | System | Set timeout budget: `timeout_secs * min(max_tool_iterations, 4)` | — | — |
| 10 | System | Run `run_tool_call_loop()` with draft streaming | See Workflow 1 step 7 | — |
| 11 | System | Cancel typing indicator | — | — |
| 12 | System | Process result (success/cancel/overflow/timeout/error) | See result handling below | — |

### 4.3 Result Handling Matrix

| Outcome | Reaction | Message Sent | History Updated | Special Action |
|---|---|---|---|---|
| **Success** | ✅ | Final response (capped at 20K chars) via draft finalize or new message | Assistant response appended | Auto-save to memory |
| **Cancellation** | — | None | — | Prior task aborted for new message |
| **Context overflow** | ⚠️ | `"[Context limit reached, history compacted]"` | History compacted: last 12 turns, 600 chars each | `compact_sender_history()` |
| **Timeout** | ⚠️ | Timeout error message | `[Task timed out after Ns]` appended | Budget = timeout × min(iters, 4) |
| **Agent error** | ⚠️ | Sanitized error message | — | Error paths stripped, credentials removed |

### 4.4 Draft Streaming (Channels Supporting Live Updates)

```
Agent Loop                    Draft Updater Task              Channel
    │                              │                            │
    │── on_delta("Thinking...")──►│                            │
    │                              │── send_draft("...") ─────►│
    │── on_delta("chunk1") ──────►│                            │
    │── on_delta("chunk2") ──────►│                            │
    │                              │── [≥80 chars accumulated]  │
    │                              │── update_draft(text) ─────►│
    │── on_delta("chunk3") ──────►│                            │
    │                              │── update_draft(text) ─────►│
    │── on_delta(CLEAR_SENTINEL)─►│                            │
    │                              │── [reset accumulated text] │
    │── on_delta("final") ───────►│                            │
    │                              │── finalize_draft(text) ───►│
```

`CLEAR_SENTINEL = "\x00CLEAR\x00"` — sent when tool results arrive (clears "Thinking..." or partial text before new LLM response).

---

## 5. History Compaction Sub-Flow

Triggered when conversation history exceeds `max_history_messages` (default: 50).

| Step | Actor | Action | System Response | Data Updated |
|---|---|---|---|---|
| 1 | System | Count non-system messages; if ≤ 20, skip | — | — |
| 2 | System | Collect oldest messages up to 12,000 chars total | — | — |
| 3 | System | Call LLM: `"Summarize the following conversation history in max 12 bullet points."` (temperature 0.2) | Summary text (max 2,000 chars) | Cost tracker |
| 4 | System | Replace old messages with `[Compacted history]\n{summary}` as system message | — | History |
| 5 | System | On LLM failure: fallback to keeping last 20 messages, dropping older ones | — | History |

---

## 6. Memory Hygiene Sub-Flow

Background maintenance triggered by the memory backend.

| Step | Actor | Action | System Response | Data Updated |
|---|---|---|---|---|
| 1 | System | Check `hygiene_enabled` (default: true) | — | — |
| 2 | System | Archive entries older than `archive_after_days` (7) | Move category to `archived` | Memory DB |
| 3 | System | Purge entries older than `purge_after_days` (30) | Delete entries | Memory DB |
| 4 | System | Purge conversation entries older than `conversation_retention_days` (30) | Delete conversation-category entries | Memory DB |
| 5 | System | If `snapshot_on_hygiene` enabled: write `MEMORY_SNAPSHOT.md` | Markdown export of all core memories | Snapshot file |

---

## 7. Cold-Boot Hydration Sub-Flow

Triggered on startup when memory database is empty.

| Step | Actor | Action | System Response | Data Updated |
|---|---|---|---|---|
| 1 | System | Open/create `brain.db` | Fresh database with schema applied | Memory DB |
| 2 | System | Check `auto_hydrate` (default: true) | — | — |
| 3 | System | Check if `MEMORY_SNAPSHOT.md` exists in workspace | — | — |
| 4 | System | If snapshot exists and DB is empty: parse markdown entries | Extract key-value pairs from snapshot format | — |
| 5 | System | Store each entry as `Core` category memory | `memory.store(key, content, Core)` per entry | Memory DB |

---

*Next file: `04_API_and_Integrations.md` — Endpoint catalog, agent tool specifications, and integration interfaces.*
