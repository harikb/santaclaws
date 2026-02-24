# 03 — Core Workflows

> Step-by-step user journeys for the three primary workflows.
> Format: **Actor → Action → System Response → Data Updated**

---

## Workflow 1: Inbound Message → Response Delivery

The most complex and central workflow. A message arrives from any channel, is routed to an agent, processed through an LLM with iterative tool execution, and the response is delivered back.

### 1.1 Sequence Diagram

```
User        Channel       Bus         Router       Agent        LLM         Tools       Session
 │            │            │            │            │            │            │            │
 │──message──▶│            │            │            │            │            │            │
 │            │──publish──▶│            │            │            │            │            │
 │            │  Inbound   │            │            │            │            │            │
 │            │            │──consume──▶│            │            │            │            │
 │            │            │            │──resolve──▶│            │            │            │
 │            │            │            │ (7-level)  │            │            │            │
 │            │            │            │◀─agentID───│            │            │            │
 │            │            │            │            │            │            │            │
 │            │            │            │──────────▶ │            │            │            │
 │            │            │            │            │──build     │            │            │
 │            │            │            │            │  system    │            │         ┌──│
 │            │            │            │            │  prompt───────────────────────────│  │
 │            │            │            │            │            │            │  history │  │
 │            │            │            │            │◀───────────────────────────────────│  │
 │            │            │            │            │            │            │         └──│
 │            │            │            │            │──Chat()──▶ │            │            │
 │            │            │            │            │            │            │            │
 │            │            │            │            │◀─response──│            │            │
 │            │            │            │            │  (tool_calls)           │            │
 │            │            │            │            │            │            │            │
 │            │            │            │            │──Execute()─────────────▶│            │
 │            │            │            │            │◀─ToolResult─────────────│            │
 │            │            │            │            │            │            │            │
 │            │            │            │            │──Chat()──▶ │  (loop)    │            │
 │            │            │            │            │◀─response──│            │            │
 │            │            │            │            │  (no tools = done)      │            │
 │            │            │            │            │            │            │         ┌──│
 │            │            │            │            │──save──────────────────────────────│  │
 │            │            │            │            │            │            │         └──│
 │            │            │◀────────── │            │            │            │            │
 │            │  Outbound  │            │            │            │            │            │
 │            │◀──dispatch─│            │            │            │            │            │
 │◀──reply────│            │            │            │            │            │            │
```

### 1.2 Detailed Steps

#### Step 1 — Channel Receives Message

| | Detail |
|---|---|
| **Actor** | External user on a messaging platform |
| **Action** | Sends a message (text, media, or command) to the bot |
| **System Response** | Channel adapter parses the platform-specific payload into an `InboundMessage` |
| **Data Updated** | None |

**Access check:** The channel calls `IsAllowed(senderID)` against the `allow_from` list. If the list is non-empty and the sender is not in it, the message is silently dropped.

**InboundMessage fields populated:**
- `channel` — adapter name (e.g., `"telegram"`)
- `sender_id` — platform user ID
- `chat_id` — platform chat/group ID
- `content` — message text
- `media` — attachment URLs (if any)
- `metadata` — platform-specific fields: `account_id`, `peer_kind`, `peer_id`, `parent_peer_kind`, `parent_peer_id`, `guild_id`, `team_id`

---

#### Step 2 — Message Published to Bus

| | Detail |
|---|---|
| **Actor** | Channel adapter |
| **Action** | Calls `bus.PublishInbound(msg)` |
| **System Response** | Message enters the inbound buffered channel (cap 100). Non-blocking unless buffer is full. |
| **Data Updated** | Bus inbound queue |

---

#### Step 3 — Agent Loop Consumes Message

| | Detail |
|---|---|
| **Actor** | Agent Loop (goroutine running `Run()`) |
| **Action** | Calls `bus.ConsumeInbound(ctx)` — blocks until message available or context cancelled |
| **System Response** | Returns the next `InboundMessage` from the queue |
| **Data Updated** | Message removed from inbound queue |

**Branching:**
- If `channel == "system"` → route to `processSystemMessage()` (handles subagent completions)
- If content is a command (`/show`, `/list`, `/switch`) → handle directly, skip LLM
- Otherwise → proceed to routing

---

#### Step 4 — Route Resolution (7-Level Cascade)

| | Detail |
|---|---|
| **Actor** | Route Resolver |
| **Action** | Evaluates all configured bindings against the message metadata |
| **System Response** | Returns a `ResolvedRoute` with `agentID`, `sessionKey`, and `matchedBy` |
| **Data Updated** | None |

**Cascade evaluation order:**

1. Filter bindings to those matching `channel` and `accountID`
2. Check peer ID match → `matchedBy: "binding.peer"`
3. Check parent peer ID match → `matchedBy: "binding.peer.parent"`
4. Check guild ID match → `matchedBy: "binding.guild"`
5. Check team ID match → `matchedBy: "binding.team"`
6. Check non-wildcard account match → `matchedBy: "binding.account"`
7. Check wildcard (`*`) account match → `matchedBy: "binding.channel"`
8. Fall through → `matchedBy: "default"`, use default agent

**Session key construction** depends on `dm_scope` config:
- `main` → `agent:{agentId}:main` (all DMs share one session)
- `per-peer` → `agent:{agentId}:direct:{peerId}`
- `per-channel-peer` → `agent:{agentId}:{channel}:direct:{peerId}`
- `per-account-channel-peer` → `agent:{agentId}:{channel}:{accountId}:direct:{peerId}`

For group messages, the key always includes the group ID: `agent:{agentId}:{channel}:group:{groupId}`

---

#### Step 5 — System Prompt Assembly

| | Detail |
|---|---|
| **Actor** | ContextBuilder |
| **Action** | Calls `BuildMessages()` to construct the full message array for the LLM |
| **System Response** | Returns `[system_prompt, ...history_messages, user_message]` |
| **Data Updated** | None (read-only) |

**System prompt is assembled in order:**

1. **Identity block** — current datetime, OS/arch, Go version, workspace path
2. **Tools section** — `"## Available Tools"` with dynamic tool summaries + instruction: `"You MUST use tools to perform actions"`
3. **Bootstrap files** (read from workspace, skipped if missing):
   - `AGENTS.md` — agent behavior guidelines
   - `SOUL.md` — personality definition
   - `USER.md` — user context
   - `IDENTITY.md` — agent identity
4. **Skills summary** — list of available skills with descriptions, instruction to `read_file SKILL.md`
5. **Memory context**:
   - `MEMORY.md` — long-term memory (full contents)
   - Last 3 daily notes (`YYYYMMDD.md`) — full contents
6. **Session info** — `"Channel: {channel}\nChat ID: {chatID}"`
7. **Conversation summary** (if exists) — `"## Summary of Previous Conversation\n\n{summary}"`

All sections joined with `"\n\n---\n\n"`.

**History sanitization** before inclusion:
- Orphaned tool-result messages (no preceding assistant with tool_calls) are dropped
- Assistant messages with tool_calls that appear at invalid positions are dropped
- Prevents LLM API errors from malformed conversation sequences

---

#### Step 6 — Save User Message

| | Detail |
|---|---|
| **Actor** | Agent Loop |
| **Action** | `agent.Sessions.AddMessage(sessionKey, "user", content)` |
| **System Response** | Message appended to session history |
| **Data Updated** | Session (in-memory) |

---

#### Step 7 — LLM Iteration Loop

| | Detail |
|---|---|
| **Actor** | Agent Loop |
| **Action** | Enters `runLLMIteration()` — iterates up to `max_tool_iterations` times (default 20) |
| **System Response** | Each iteration: call LLM → if tool_calls, execute tools → feed results back → repeat |
| **Data Updated** | Session history (assistant messages, tool results appended each iteration) |

**Per-iteration detail:**

**7a. Call LLM:**
- Build tool definitions from registry (`ToProviderDefs()`)
- Call `provider.Chat(ctx, messages, toolDefs, model, {max_tokens, temperature})`
- If fallback candidates configured, use `fallback.Execute()` to try alternatives on retriable errors

**7b. Context overflow retry (up to 2 retries):**
- If LLM returns a token/context/length error:
  - First retry: notify user `"Context window exceeded. Compressing history and retrying..."`
  - Call `forceCompression()` — drops oldest 50% of messages from history
  - Rebuild messages and retry
- Non-context errors: fail immediately

**7c. Check finish reason:**
- `finish_reason = "stop"` or `"length"` → no tool calls → set `finalContent = response.Content` → **exit loop**
- `finish_reason = "tool_calls"` → tool calls present → continue to 7d

**7d. Save assistant message:**
- Create assistant message with `role: "assistant"`, `content`, and all `tool_calls`
- Append to messages array and session history

**7e. Execute each tool call sequentially:**
- For each tool call in the response:
  1. Parse `function.name` and `function.arguments` (JSON)
  2. Call `agent.Tools.ExecuteWithContext(ctx, name, args, channel, chatID, asyncCallback)`
  3. If the tool returns `ForUI` content → publish directly to user (immediate partial output)
  4. Build tool result message: `role: "tool"`, `content: ForLLM`, `tool_call_id: tc.ID`
  5. Append tool result to messages array and session history

**7f. Loop back to 7a** with updated messages (including tool results)

**Loop termination conditions:**
- LLM returns no tool calls → natural completion
- Iteration count reaches `max_tool_iterations` → forced stop, deliver partial content
- Context error after max retries → error returned
- Unrecoverable LLM error → error returned

---

#### Step 8 — Save Response and Session

| | Detail |
|---|---|
| **Actor** | Agent Loop |
| **Action** | Save assistant response to session, persist to disk |
| **System Response** | `AddMessage(sessionKey, "assistant", finalContent)` then `Save(sessionKey)` — atomic write |
| **Data Updated** | Session file on disk (`sessions/{key}.json`) |

If `finalContent` is empty, substitute `DefaultResponse`: `"I've completed processing but have no response to give."`

---

#### Step 9 — Optional Summarization

| | Detail |
|---|---|
| **Actor** | Agent Loop |
| **Action** | `maybeSummarize()` — checks if session needs compression |
| **System Response** | If triggered, spawns async goroutine to summarize older messages |
| **Data Updated** | Session summary and truncated history |

**Trigger conditions** (any one):
- History exceeds 20 messages
- Estimated token count exceeds 75% of context window

**Summarization process:**
1. Keep the last 4 messages for continuity
2. Take messages 0 through `len-5` for summarization
3. Skip individual messages exceeding `context_window / 2` tokens
4. If more than 10 messages: split in half, summarize each part, merge summaries
5. If 10 or fewer: summarize in one pass
6. Replace old history with summary + kept messages
7. Save updated session

**Token estimation heuristic:** `totalChars × 2 / 5` (assumes ~2.5 characters per token)

---

#### Step 10 — Response Delivery

| | Detail |
|---|---|
| **Actor** | Agent Loop → Bus → Channel Manager |
| **Action** | Publish `OutboundMessage` to bus outbound channel |
| **System Response** | Channel manager's outbound dispatcher routes to the correct channel adapter |
| **Data Updated** | Bus outbound queue, then delivered to platform |

**Deduplication guard:** If a tool already sent a message during execution (tracked via `HasSentInRound()`), the final response is not re-published. This prevents the user from seeing duplicate messages.

**State update:** If the channel and chatID are non-empty and not internal, record as last-used: `stateManager.SetLastChannel("channel:chatID")`

---

### 1.3 Error Handling Summary

| Error | Handling | User Impact |
|---|---|---|
| Sender not in allow_from | Silent drop at channel level | No response |
| No agent found for route | Falls back to default agent | Transparent |
| LLM context overflow | Compress history (drop oldest 50%), retry up to 2× | User sees "Compressing history..." notification |
| LLM retriable error (rate limit, timeout, overload) | Try fallback models in order | Transparent (may see different model behavior) |
| LLM non-retriable error (auth, billing, format) | Fail immediately | User sees error message |
| Tool execution error | Error text returned to LLM as tool result; LLM decides how to proceed | LLM may retry or explain the failure |
| Max iterations reached | Loop exits, partial content delivered | May receive incomplete answer |
| Empty LLM response | Substitute default message | User sees "I've completed processing but have no response to give." |

---

## Workflow 2: Skill Discovery and Installation

A two-step workflow: the agent searches for skills on the ClawHub registry, then installs a selected skill into the workspace.

### 2.1 Sequence Diagram

```
Agent/LLM     find_skills     Cache       ClawHub       install_skill    Filesystem
    │              │            │            │                │               │
    │──search()───▶│            │            │                │               │
    │              │──Get()────▶│            │                │               │
    │              │            │            │                │               │
    │              │  hit?──yes─│─format────▶│                │               │
    │              │            │ (return)   │                │               │
    │              │  miss──────│            │                │               │
    │              │──SearchAll()───────────▶│                │               │
    │              │◀──results───────────────│                │               │
    │              │──Put()────▶│            │                │               │
    │◀─results─────│            │            │                │               │
    │              │            │            │                │               │
    │──install()──────────────────────────────────────────────▶               │
    │              │            │            │                │──validate──   │
    │              │            │            │                │  slug         │
    │              │            │            │                │──check        │
    │              │            │            │                │  exists──────▶│
    │              │            │            │◀──GetMeta()────│               │
    │              │            │            │──meta──────────▶               │
    │              │            │            │◀──Download()───│               │
    │              │            │            │──ZIP───────────▶               │
    │              │            │            │                │──extract─────▶│
    │              │            │            │                │──malware      │
    │              │            │            │                │  check        │
    │              │            │            │                │──write        │
    │              │            │            │                │  origin──────▶│
    │◀─success────────────────────────────────────────────────│               │
```

### 2.2 Step A: Skill Search (`find_skills` tool)

#### Step A1 — Validate and Normalize Query

| | Detail |
|---|---|
| **Actor** | LLM (via tool call) |
| **Action** | Calls `find_skills` with `{query: "weather", limit: 5}` |
| **System Response** | Normalize query: `strings.ToLower(strings.TrimSpace(query))`. Validate non-empty. Clamp limit to [1, 20]. |
| **Data Updated** | None |

---

#### Step A2 — Cache Lookup

| | Detail |
|---|---|
| **Actor** | find_skills tool |
| **Action** | `cache.Get(normalizedQuery)` |
| **System Response** | Two-phase lookup: (1) exact match by key, (2) trigram-based similarity match (Jaccard >= 0.7). Entries with expired TTL (3600s) are skipped. |
| **Data Updated** | LRU order updated on hit |

**If cache hit:** Format results and return immediately — no API call.

**Trigram similarity:** Queries are decomposed into overlapping 3-character windows. Jaccard similarity = `|A ∩ B| / |A ∪ B|`. Threshold: 0.7 (70% similar).

---

#### Step A3 — Registry Search (Cache Miss)

| | Detail |
|---|---|
| **Actor** | find_skills tool |
| **Action** | `registryMgr.SearchAll(ctx, query, limit)` — queries all enabled registries |
| **System Response** | ClawHub registry: HTTP GET to `{baseURL}{searchPath}?q={query}&limit={limit}` |
| **Data Updated** | None |

**ClawHub response parsing:**
- Parse JSON response with `Results[]` array
- For each result: extract `Score`, `Slug`, `DisplayName`, `Summary`, `Version`
- Skip entries with empty slug or summary
- Tag each with `RegistryName: "clawhub"`

---

#### Step A4 — Cache Store

| | Detail |
|---|---|
| **Actor** | find_skills tool |
| **Action** | `cache.Put(query, results)` if results non-empty |
| **System Response** | LRU eviction if at max capacity (100 entries). Build trigrams for similarity matching. |
| **Data Updated** | Search cache (in-memory) |

---

#### Step A5 — Format and Return

| | Detail |
|---|---|
| **Actor** | find_skills tool |
| **Action** | Build formatted text output |
| **System Response** | Returns `SilentResult` (not shown to user, only to LLM) with: count, slug, version, score, registry, display name, summary for each result |
| **Data Updated** | None |

---

### 2.3 Step B: Skill Installation (`install_skill` tool)

#### Step B1 — Acquire Lock and Validate

| | Detail |
|---|---|
| **Actor** | LLM (via tool call) |
| **Action** | Calls `install_skill` with `{slug: "weather", registry: "clawhub"}` |
| **System Response** | Acquire workspace-level mutex (prevents concurrent installs). Validate slug against pattern `^[a-zA-Z0-9]+(-[a-zA-Z0-9]+)*$`. Validate registry name similarly. |
| **Data Updated** | None |

---

#### Step B2 — Check Existing Installation

| | Detail |
|---|---|
| **Actor** | install_skill tool |
| **Action** | `os.Stat(workspace/skills/{slug})` |
| **System Response** | If directory exists and `force=false` → return error "already installed". If `force=true` → `os.RemoveAll()` existing directory. |
| **Data Updated** | Filesystem (if force-removing) |

---

#### Step B3 — Fetch Metadata

| | Detail |
|---|---|
| **Actor** | ClawHub registry |
| **Action** | HTTP GET to `{baseURL}{skillsPath}/{slug}` |
| **System Response** | Returns `SkillMeta`: slug, display name, summary, latest version, moderation flags |
| **Data Updated** | None |

**Graceful fallback:** If metadata fetch fails, installation continues without metadata (version defaults to `"latest"`).

---

#### Step B4 — Download ZIP Archive

| | Detail |
|---|---|
| **Actor** | ClawHub registry |
| **Action** | HTTP GET to `{baseURL}{downloadPath}?slug={slug}&version={version}` with optional Bearer token |
| **System Response** | Streams response body in ~32KB chunks to a temp file. Enforces `maxZipSize` (default 50 MB). |
| **Data Updated** | Temp file on disk |

---

#### Step B5 — Extract to Workspace

| | Detail |
|---|---|
| **Actor** | install_skill tool |
| **Action** | `utils.ExtractZipFile(tmpPath, targetDir)` |
| **System Response** | Extracts all ZIP contents to `{workspace}/skills/{slug}/` |
| **Data Updated** | Skill files on disk |

**Cleanup on failure:** If extraction fails, `os.RemoveAll(targetDir)` is called and the error is returned.

---

#### Step B6 — Malware Check

| | Detail |
|---|---|
| **Actor** | install_skill tool |
| **Action** | Check `result.IsMalwareBlocked` from metadata |
| **System Response** | If flagged: `os.RemoveAll(targetDir)` and return error "flagged as malicious" |
| **Data Updated** | Skill directory removed if flagged |

---

#### Step B7 — Write Origin Metadata

| | Detail |
|---|---|
| **Actor** | install_skill tool |
| **Action** | Write `.skill-origin.json` to the skill directory |
| **System Response** | JSON file with: `version: 1`, `registry`, `slug`, `installed_version`, `installed_at` (Unix ms) |
| **Data Updated** | `.skill-origin.json` on disk |

---

#### Step B8 — Return Result

| | Detail |
|---|---|
| **Actor** | install_skill tool |
| **Action** | Build output string |
| **System Response** | Returns `SilentResult` with: success message, version, location, description. Prepends warning if `IsSuspicious`. |
| **Data Updated** | None (skill is now available on next prompt assembly) |

---

### 2.4 Error Handling Summary

| Error | Handling | Outcome |
|---|---|---|
| Empty query | Immediate error return | LLM told to provide a query |
| Invalid slug/registry name | Validation error | LLM told the identifier is invalid |
| Skill already installed | Error (unless `force=true`) | LLM informed; can retry with force |
| Registry not found | Error return | LLM told registry doesn't exist |
| HTTP error from ClawHub | Error with status code and body | LLM sees the HTTP error |
| ZIP exceeds 50 MB | Download aborted | Error returned |
| Extraction failure | `os.RemoveAll(targetDir)` cleanup | Error returned |
| Malware flag | `os.RemoveAll(targetDir)` cleanup | Error: "flagged as malicious" |
| Metadata fetch failure | Graceful fallback (continue without metadata) | Installation proceeds with version "latest" |

---

## Workflow 3: Gateway Lifecycle

The gateway is the primary production runtime — a long-running process that manages all channels, services, and the agent loop.

### 3.1 Startup Sequence Diagram

```
                    ┌─────────────────────────────────────────────┐
                    │            Gateway Startup                   │
                    └─────────────────────────────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
        Load Config            Create Provider          Create Bus
              │                       │                       │
              └───────────┬───────────┘                       │
                          ▼                                   │
                    Create AgentLoop ◀────────────────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        Setup Cron   Setup Heartbeat  Create Channel Manager
              │           │           │
              │           │           ├── Setup Voice (optional)
              │           │           │
              ▼           ▼           ▼
        Start Cron   Start Heartbeat  Start Channels
              │           │           │
              └───────────┼───────────┘
                          ▼
                    Start Health Server (:18790)
                          │
                          ▼
                    Start Agent Loop (goroutine)
                          │
                          ▼
                    ═══ RUNNING ═══
                    (await SIGINT)
                          │
                          ▼
                    Graceful Shutdown
```

### 3.2 Detailed Steps

#### Step 1 — Load Configuration

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | `loadConfig()` — reads `~/.picoclaw/config.json`, applies env var overrides |
| **System Response** | Returns merged `Config` struct. Exits with code 1 on failure. |
| **Data Updated** | None |

---

#### Step 2 — Create LLM Provider

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | `providers.CreateProvider(cfg)` — resolves and instantiates the primary LLM provider |
| **System Response** | Returns `LLMProvider` interface. The resolved model ID is written back to `cfg.Agents.Defaults.Model`. Exits with code 1 on failure. |
| **Data Updated** | Config (model field updated to resolved value) |

---

#### Step 3 — Create Message Bus

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | `bus.NewMessageBus()` |
| **System Response** | Creates two buffered channels: `inbound[100]`, `outbound[100]`. Initializes empty handler map. |
| **Data Updated** | None |

---

#### Step 4 — Create Agent Loop

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | `agent.NewAgentLoop(cfg, msgBus, provider)` |
| **System Response** | Creates `AgentRegistry` (instantiates all configured agents with defaults, creates implicit "main" if none configured). Registers shared tools: web search, message, spawn, subagent, skills. |
| **Data Updated** | Tool registry populated |

---

#### Step 5 — Setup Cron Service

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | `setupCronTool()` — creates CronService with store at `{workspace}/cron/jobs.json` |
| **System Response** | Creates `CronTool`, registers it in the agent loop's tool registry. Sets job handler that delegates to `cronTool.ExecuteJob()`. |
| **Data Updated** | Tool registry (cron tool added) |

---

#### Step 6 — Setup Heartbeat Service

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | `heartbeat.NewHeartbeatService(workspace, interval, enabled)` |
| **System Response** | Creates heartbeat service with bus reference and handler callback. Handler uses `agentLoop.ProcessHeartbeat()` to run a periodic agent turn with a heartbeat prompt. |
| **Data Updated** | None |

---

#### Step 7 — Create Channel Manager

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | `channels.NewManager(cfg, msgBus)` |
| **System Response** | Iterates all channel configs. For each enabled channel with valid credentials, instantiates the adapter. Exits with code 1 on failure. Injected into AgentLoop. |
| **Data Updated** | None |

---

#### Step 8 — Setup Voice Transcription (Optional)

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | Look for Groq API key in config or model_list. If found, create `voice.NewGroqTranscriber(key)`. |
| **System Response** | Attaches transcriber to Telegram, Discord, and Slack channels via `SetTranscriber()`. Enables voice-to-text on these channels. |
| **Data Updated** | Channel instances (transcriber field set) |

---

#### Step 9 — Start All Services

| | Detail |
|---|---|
| **Actor** | Gateway process |
| **Action** | Sequential start of: cron → heartbeat → devices → channels → health server → agent loop |
| **System Response** | Each service starts in order. Service start errors are logged but do not block other services (non-fatal). |
| **Data Updated** | All services running |

**Service start details:**

| Service | Start Method | Failure Handling |
|---|---|---|
| Cron | `cronService.Start()` | Log error, continue |
| Heartbeat | `heartbeatService.Start()` | Log error, continue |
| Devices | `deviceService.Start(ctx)` | Log error, continue |
| Channels | `channelManager.StartAll(ctx)` | Log error, continue |
| Health | `healthServer.Start()` (goroutine) | Log if not `ErrServerClosed` |
| Agent Loop | `agentLoop.Run(ctx)` (goroutine) | Runs until context cancelled |

**Health server endpoints:**
- `GET /health` → always 200 (liveness)
- `GET /ready` → 200 if ready and all checks pass, 503 otherwise

---

#### Step 10 — Await Shutdown Signal

| | Detail |
|---|---|
| **Actor** | OS |
| **Action** | `signal.Notify(sigChan, os.Interrupt)` — blocks on `<-sigChan` |
| **System Response** | Gateway is fully operational. Processing messages, running cron jobs, heartbeating. |
| **Data Updated** | Ongoing (sessions, state, memory) |

---

#### Step 11 — Graceful Shutdown

| | Detail |
|---|---|
| **Actor** | Gateway process (on SIGINT) |
| **Action** | Ordered shutdown of all services |
| **System Response** | Each service stopped in reverse-dependency order |
| **Data Updated** | Final state flushed to disk |

**Shutdown order:**

1. `cancel()` — signal all goroutines via context
2. `healthServer.Stop(ctx)` — stop accepting health checks
3. `deviceService.Stop()` — stop device monitoring
4. `heartbeatService.Stop()` — stop heartbeat timer
5. `cronService.Stop()` — stop cron scheduler
6. `agentLoop.Stop()` — set running flag to false
7. `channelManager.StopAll(ctx)` — disconnect all channels

---

### 3.3 Gateway vs. Single-Shot Agent Mode

| Aspect | Gateway (`picoclaw gateway`) | Agent (`picoclaw agent -m "..."`) |
|---|---|---|
| **Lifetime** | Long-running (until SIGINT) | Single message, then exit |
| **Channels** | All enabled channels started | None — CLI only |
| **Message Bus** | Active (inbound/outbound routing) | Created but not consumed |
| **Cron** | Started and scheduling | Not available |
| **Heartbeat** | Running at configured interval | Not available |
| **Health Server** | Listening on configured port | Not available |
| **Voice** | Attached to channels | Not available |
| **Session Persistence** | Continuous across messages | Loaded and saved for one turn |
| **Entry Point** | `agentLoop.Run(ctx)` (goroutine, loop) | `agentLoop.ProcessDirect(ctx, msg, key)` (single call) |
| **Interactive Mode** | N/A | Available if no `-m` flag (readline REPL) |

---

### 3.4 Error Handling Summary

| Error | Phase | Handling | Impact |
|---|---|---|---|
| Config load failure | Startup | `os.Exit(1)` | Gateway does not start |
| Provider creation failure | Startup | `os.Exit(1)` | Gateway does not start |
| Channel manager creation failure | Startup | `os.Exit(1)` | Gateway does not start |
| Individual channel start failure | Startup | Logged, other channels continue | Partial functionality |
| Cron/Heartbeat/Device start failure | Startup | Logged, gateway continues | Feature unavailable |
| Health server error | Runtime | Logged (unless `ErrServerClosed`) | Health checks unavailable |
| Message processing error | Runtime | Error text sent as response to user | User sees error |
| Bus buffer full | Runtime | Publisher blocks until space available | Message delay |
| Channel send failure | Runtime | Logged, message lost | User does not receive response |
