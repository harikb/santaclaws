# 03 — Core Workflows

## 3.1 Inbound Message Routing

The most complex and performance-critical workflow. Traces a message from arrival on a channel through security checks, agent execution, and reply delivery.

### Sequence Diagram

```
Sender        Channel Plugin     Security       Router        Queue         Agent Engine     Outbound
  │               │                │              │             │               │              │
  │──message──▶   │                │              │             │               │              │
  │               │ extract msg    │              │             │               │              │
  │               │ download media │              │             │               │              │
  │               │ debounce       │              │             │               │              │
  │               │────────────────▶              │             │               │              │
  │               │                │ DM policy?   │             │               │              │
  │               │                │ allowlist?    │             │               │              │
  │               │                │ group policy? │             │               │              │
  │               │                │ mention gate? │             │               │              │
  │               │                │──────────────▶│             │               │              │
  │               │                │              │ match binding│              │              │
  │               │                │              │ build key   │               │              │
  │               │                │              │─────────────▶│              │              │
  │               │                │              │             │ check active  │              │
  │               │                │              │             │ queue mode    │              │
  │               │                │              │             │───────────────▶│             │
  │               │                │              │             │               │ build prompt │
  │               │                │              │             │               │ call LLM     │
  │               │                │              │             │               │ run tools    │
  │               │                │              │             │               │──────────────▶
  │               │                │              │             │               │             │ chunk text
  │               │                │              │             │               │             │ format
  │◀─────────────────────────────────────────────────────────────────────────────────────────│ deliver
```

### Step-by-Step Flow

#### Step 1: Channel Receives Message

**Actor:** External messaging platform (e.g., Telegram Bot API)
**Entry:** Channel plugin listener (e.g., grammY `bot.on("message")`)

| Action | Detail |
|---|---|
| Skip duplicate | Checks update ID against persistent watermark |
| Extract message data | Sender ID/username, text, caption, entities, timestamp, reply context, forward origin |
| Download media | Calls platform API to download files; saves to local path with detected MIME type |
| Buffer fragments | Telegram splits large pastes (~4096 chars) into multiple updates; buffers up to 12 parts / 50K chars within 1.5s gap |
| Buffer media groups | Multi-photo messages buffered for 500ms, then flushed as combined context |
| Debounce rapid messages | Configurable debounce window; flushes accumulated text as single context |

**Data produced:** `MsgContext` with `Body`, `BodyForAgent`, `From`, `To`, `SessionKey`, `ChatType`, `MediaPath`, `MediaType`, sender metadata.

#### Step 2: Access Control

**Actor:** Security layer (`buildTelegramMessageContext()`)

**DM decision tree:**

```
dmPolicy = config.channels.<channel>.dmPolicy ?? "pairing"

if "disabled"  → DROP (silent)
if "open"      → ALLOW
if "pairing":
    sender in allowFrom?  → ALLOW
    else → generate 8-char code, send to sender, DROP
if "allowlist":
    sender in allowFrom?  → ALLOW
    else → DROP (silent)
```

**Group decision tree:**

```
groupPolicy = resolveChannelGroupPolicy(channel, guildId)

if "disabled"  → DROP
if "allowlist":
    group in groups config?  → check per-group allowFrom
    sender authorized?       → continue to mention gate
    else → DROP
if "open" → continue to mention gate

Mention gate:
    requireMention = config or session override
    wasMentioned = @bot in entities OR mention regex match OR reply to bot
    if requireMention && !wasMentioned → RECORD to group history, DROP
```

**AllowFrom resolution:** Merges config `allowFrom[]` + disk store (pairing-approved senders). Store-based entries are excluded when policy is `"allowlist"` (config-only mode).

**Data updated:** Session `lastFrom`, `lastProvider`, `lastSeen` persisted via fire-and-forget `recordInboundSession()`.

#### Step 3: Agent Route Resolution

**Actor:** Router (`resolveAgentRoute()`)

**Input:** `{ channel, accountId, peer: { kind, id }, parentPeer?, guildId?, teamId?, memberRoleIds[] }`

**Algorithm:** Waterfall through binding tiers (pre-filtered by channel + accountId, cached in WeakMap):

| Priority | Match | Example |
|---|---|---|
| 1 | `binding.peer` (exact kind + id) | Route Telegram user 12345 → agent "personal" |
| 2 | `binding.peer.parent` (thread parent) | Forum topic inherits from channel binding |
| 3 | `binding.guild+roles` | Discord guild + member role → agent "support" |
| 4 | `binding.guild` | Any member of Discord guild → agent "team" |
| 5 | `binding.team` | Slack team → agent "work" |
| 6 | `binding.account` (specific) | Specific bot account → agent "bot-2" |
| 7 | `binding.channel` (wildcard `*`) | All Telegram → agent "telegram-default" |
| 8 | `default` | Global fallback (typically `"main"`) |

**Session key construction** (based on `session.dmScope`):

| Scope | Key Pattern | Use Case |
|---|---|---|
| `main` (default) | `agent:main:main` | Single shared conversation |
| `per-peer` | `agent:main:direct:12345` | Per-sender isolation |
| `per-channel-peer` | `agent:main:telegram:direct:12345` | Per-channel per-sender |
| `per-account-channel-peer` | `agent:main:telegram:bot1:direct:12345` | Full isolation |

**Output:** `ResolvedAgentRoute { agentId, channel, accountId, sessionKey, mainSessionKey, matchedBy }`

#### Step 4: Queue / Concurrency Control

**Actor:** Command queue + session-level concurrency guard

The system checks if an agent turn is already active for this session (`isEmbeddedPiRunActive(sessionId)`). If active, the queue mode determines behavior:

| Queue Mode | Behavior |
|---|---|
| `none` | Drop the message (no queue) |
| `followup` | Enqueue; run after current turn finishes |
| `collect` | Accumulate messages; run as single combined turn |
| `interrupt` | Abort current turn (`clearCommandLane` + `abortEmbeddedPiRun`), start new |
| `steer` | Inject into running agent via `queueEmbeddedPiMessage()` |
| `steer-backlog` | Steer if streaming; followup otherwise |
| `steer+backlog` | Steer and also enqueue followup |
| `queue` | Standard FIFO queue |

If no turn is active, the message proceeds immediately through the `main` command lane (serial, max concurrency 1).

#### Step 5: Agent Turn Execution

**Actor:** Agent engine (`getReplyFromConfig` → `runPreparedReply` → `runReplyAgent` → `runAgentTurnWithFallback`)

**Pre-turn enrichment:**

| Step | Function | Effect |
|---|---|---|
| Media understanding | `applyMediaUnderstanding()` | Vision/transcription of images, audio, video → text injected into `BodyForAgent` |
| Link understanding | `applyLinkUnderstanding()` | URL content extraction → appended to context |
| Session init | `initSessionState()` | Load session store, detect new/reset, set `systemSent` flag |
| Directive parsing | `resolveReplyDirectives()` | Process inline commands: `/think high`, `/model gpt-4o`, `/new`, `/verbose` |
| Inline actions | `handleInlineActions()` | Handle `/status`, `/help`, `/compact` — return early without LLM call |

**System prompt construction:**

```
[Agent identity + workspace context + skill instructions]
  + buildInboundMetaSystemPrompt()     — channel/sender/timestamp context
  + buildGroupChatContext()            — group name, participants
  + buildGroupIntro()                  — first-turn activation behavior
  + per-group/topic custom system prompt
  + memory search results (if enabled)
```

**User message construction:**

```
[Thread context note (if in thread)]
  + [Inbound user context metadata (timestamp, sender)]
  + [Actual message text]
```

**LLM dispatch:**

| Runner | Used When |
|---|---|
| `runEmbeddedPiAgent()` | Default — native API calls to Anthropic, OpenAI, Gemini, etc. |
| `runCliAgent()` | CLI-based providers (e.g., `claude-code`) |

**Pre-LLM steps:**
- `runMemoryFlushIfNeeded()` — compact session if approaching context window limit
- `ensureSkillSnapshot()` — snapshot workspace skills/tools for this turn
- `prependSystemEvents()` — inject queued system events (cron results, reactions)

**Error recovery:**

| Error | Recovery |
|---|---|
| Compaction failure | Reset session, retry with fresh context |
| Role ordering conflict | Reset session, retry |
| Context overflow | Surface error to user |
| HTTP 502/521 (transient) | Single retry after 2.5s delay |
| Model unavailable | Fallback chain: try `model.fallbacks[]` in order |

#### Step 6: Streaming / Live Updates

**Actor:** Draft stream + lane coordinator

Two independent delivery lanes operate during the turn:

| Lane | Content | Source Callback |
|---|---|---|
| `answer` | Main response text | `onPartialReply(payload)` |
| `reasoning` | Extended thinking blocks | `onReasoningStream(payload)` |

**Streaming pipeline:**

```
LLM token stream
  → onPartialReply({ text })
    → splitReasoningText(text)
      → answer lane: draftStream.update(text)   — debounced editMessageText
      → reasoning lane: draftStream.update(text)
```

Each `TelegramDraftStream`:
- `update(text)` — accumulates text, debounced send/edit via platform API
- `stop()` — flush pending content
- `forceNewMessage()` — next update creates a new message (between reasoning steps)

**Tool execution feedback:**
- `onToolResult(payload)` → `dispatcher.sendToolResult()` (when verbose mode enabled)
- Status reactions: queued → thinking → tool-specific emoji → done/error

**Reasoning coordinator** manages multi-step `<think>` blocks:
- Buffers final answer until reasoning draft is flushed
- Prevents interleaved reasoning/answer delivery

#### Step 7: Reply Delivery

**Actor:** Reply dispatcher + channel outbound adapter

**Pipeline:**

```
ReplyPayload[] from agent
  → TTS post-processing (if enabled + inbound was audio)
  → dispatcher.sendFinalReply(payload)
    → serialize via Promise chain (one at a time)
      → normalize: chunk text, apply markdown transforms
        → channel-specific formatting (e.g., markdownToTelegramHtml)
        → split at platform text limit with code-block safety
      → deliver each chunk via platform API
        → sendText / sendPhoto / sendVideo / sendAudio / sendVoice / sendDocument
      → handle parse errors: fallback to plain text
```

**Chunking rules:** Breaks at markdown-safe boundaries — respects code blocks, lists, headers. Platform-specific limits (Telegram: 4096 chars per message).

#### Step 8: Session Persistence

| When | What | How |
|---|---|---|
| Message arrival | `lastFrom`, `lastProvider`, `lastSeen` | Fire-and-forget `recordInboundSession()` |
| During steer | `updatedAt` | `touchActiveSessionEntry()` |
| Post-turn | Token usage, model, provider, context tokens, system prompt report | `persistRunSessionUsage()` |
| Post-turn | Model fallback notices, group intro flag | `updateSessionStore()` |
| Post-compaction | `compactionCount`, tokens after compaction | `incrementRunCompactionCount()` |

Session store writes use per-path in-process lock queues + filesystem-level locks. Writes are atomic (temp file + rename).

---

## 3.2 Heartbeat Lifecycle

A periodic background agent wake cycle that lets agents perform autonomous tasks without inbound messages.

### Sequence Diagram

```
Timer/Hook     HeartbeatRunner     Preflight        Agent Engine    Suppression     Outbound
   │                │                 │                  │              │              │
   │──interval──▶   │                 │                  │              │              │
   │  or hook       │                 │                  │              │              │
   │  or manual     │                 │                  │              │              │
   │                │──preflight──▶   │                  │              │              │
   │                │                 │ check enabled    │              │              │
   │                │                 │ check active hrs │              │              │
   │                │                 │ check queue busy │              │              │
   │                │                 │ read HEARTBEAT.md│              │              │
   │                │                 │◀────result───────│              │              │
   │                │◀────────────────│                  │              │              │
   │                │──agent turn────────────────────▶   │              │              │
   │                │                 │                  │──reply──▶    │              │
   │                │                 │                  │              │ strip token  │
   │                │                 │                  │              │ check length │
   │                │                 │                  │              │ check dup    │
   │                │                 │                  │              │──────────────▶
   │                │                 │                  │              │              │ deliver
   │                │                 │                  │              │              │ or suppress
```

### Step-by-Step Flow

#### Step 1: Scheduling and Triggers

**Actor:** Heartbeat runner + wake handler

| Trigger | Source | Priority |
|---|---|---|
| Interval timer | `heartbeat.every` config (default `"30m"`) | `INTERVAL (1)` |
| Manual wake | User command or API call | `DEFAULT (2)` |
| Hook/cron event | Plugin hook, cron job, Gmail push | `ACTION (3)` |
| Exec completion | Async shell command finished | `ACTION (3)` |
| Retry | Previous attempt skipped/failed | `RETRY (0)` |

**Coalescing:** Multiple wake requests targeting the same `agentId::sessionKey` are merged; highest priority wins. Default coalesce window: 250ms.

**Timer mechanics:** `requestHeartbeatNow()` queues a `PendingWakeReason` and schedules a `setTimeout`. Retry timers are immune to preemption by normal-priority requests.

#### Step 2: Preflight Checks

**Actor:** `runHeartbeatOnce()`

| Check | Skip Reason | Recovery |
|---|---|---|
| Heartbeat disabled globally | `"disabled"` | None (configuration change needed) |
| Heartbeat disabled for this agent | `"disabled"` | None |
| Outside active hours window | `"quiet-hours"` | Waits for next interval in active window |
| Main command lane busy | `"requests-in-flight"` | **Retry after 1 second** |
| HEARTBEAT.md empty/whitespace | `"empty-heartbeat-file"` | None (file needs content) |

Exec/cron/wake reasons **bypass** the HEARTBEAT.md file check entirely.

**Data produced:** Preflight result with `heartbeatContent` (file text), `reasonFlags`, `skipReason`.

#### Step 3: Agent Turn

**Actor:** Same agent engine as inbound messages

**Prompt construction:**
- Standard heartbeat: `"Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK."`
- Exec completion: specialized prompt relaying async command results
- Cron event: `buildCronEventPrompt(cronEvents)` with event details
- Current timestamp appended

The agent turn uses the same `getReplyFromConfig()` pipeline as inbound messages, with `isHeartbeat: true` flag.

#### Step 4: Suppression Logic

**Actor:** `normalizeHeartbeatReply()` → `stripHeartbeatToken()`

```
Agent reply text
  → strip HTML tags, &nbsp;, markdown edge chars (*, `, ~, _)
  → strip "HEARTBEAT_OK" token from start/end (loop)
  → strip up to 4 trailing punctuation chars
  → measure remaining text length

if remaining <= 300 chars (DEFAULT_HEARTBEAT_ACK_MAX_CHARS):
    → SUPPRESS (prune transcript, emit internal indicator)
if media present:
    → DELIVER (media bypasses suppression)
if text == lastHeartbeatText && within 24 hours:
    → SUPPRESS as "duplicate"
else:
    → DELIVER
```

**Transcript pruning:** When suppressed, the JSONL transcript is truncated back to the pre-heartbeat byte offset, removing the LLM turn to prevent context pollution.

#### Step 5: Delivery

**Actor:** Outbound delivery system

**Target resolution** (`resolveHeartbeatDeliveryTarget()`):

| `heartbeat.target` | Behavior |
|---|---|
| `"last"` | Use session's `lastChannel` + `lastTo` + `lastAccountId` |
| `"none"` | Silent (no delivery) |
| `"telegram"` / `"discord"` / etc. | Explicit channel with optional `heartbeat.to` |

**Visibility control** (per-channel configurable):
- `showOk: false` (default) — HEARTBEAT_OK messages are not sent
- `showAlerts: true` (default) — real content messages are sent
- `useIndicator: true` (default) — internal status events are emitted

**Delivery pipeline:** Same as inbound reply delivery — `deliverOutboundPayloads()` → channel outbound adapter → platform API.

#### Step 6: Retry and Backoff

| Condition | Action | Delay |
|---|---|---|
| `"requests-in-flight"` skip | Re-queue as retry, reschedule | 1 second |
| Exception thrown | Re-queue all pending wakes as retries | 1 second |
| Retry timer active | Normal requests cannot preempt | Until retry fires |

After successful/failed run, `advanceAgentSchedule()` computes the next interval and `scheduleNext()` sets the timer.

---

## 3.3 Onboarding & Channel Setup

The interactive wizard that bootstraps a new OpenClaw installation.

### Sequence Diagram

```
User          CLI Wizard        Config Store     Auth Provider    Channel Plugin    Gateway Service
  │               │                │                │                │                │
  │──onboard──▶   │                │                │                │                │
  │               │ risk ack       │                │                │                │
  │◀──confirm?────│                │                │                │                │
  │──yes──────▶   │                │                │                │                │
  │               │ flow select    │                │                │                │
  │◀──quick/adv?──│                │                │                │                │
  │──quickstart──▶│                │                │                │                │
  │               │────────────────────────────▶    │                │                │
  │               │                │                │ validate key   │                │
  │               │                │                │◀───────────────│                │
  │               │                │◀──write creds──│                │                │
  │               │                │                │                │                │
  │               │──pick model───▶│                │                │                │
  │               │                │◀─write model───│                │                │
  │               │                │                │                │                │
  │               │────────────────────────────────────────────▶     │                │
  │               │                │                │                │ configure      │
  │               │                │                │                │ (token/QR/     │
  │               │                │                │                │  OAuth)        │
  │               │                │◀──────write channel config──────│                │
  │               │                │                │                │                │
  │               │ DM policy      │                │                │                │
  │◀──pairing?────│                │                │                │                │
  │──yes──────▶   │                │                │                │                │
  │               │                │◀──write policy─│                │                │
  │               │                │                │                │                │
  │               │─────────────────────────────────────────────────────────────▶     │
  │               │                │                │                │                │ install
  │               │                │                │                │                │ start
  │               │                │                │                │◀───health ok───│
  │◀──ready!──────│                │                │                │                │
```

### Step-by-Step Flow

#### Step 1: Entry and Risk Acknowledgement

**Actor:** User → CLI (`openclaw onboard`)

| Action | System Response | Data Updated |
|---|---|---|
| User runs `openclaw onboard` | Displays wizard header | — |
| Security warning displayed | Multi-paragraph risk notice | — |
| User confirms `--accept-risk` or interactive confirm | Wizard proceeds | — |
| User selects flow: `quickstart` or `advanced` | Adjusts prompt depth | — |

If existing config detected: offers `keep` (resume), `modify` (edit), or `reset` (with scope: all, channels-only, model-only).

#### Step 2: Model Provider Selection

**Actor:** User → Auth choice prompt → Credential store

| Action | System Response | Data Updated |
|---|---|---|
| Provider list displayed (34+ options grouped) | Anthropic, OpenAI, Gemini, OpenRouter, Ollama, GitHub Copilot, xAI, Mistral, custom, etc. | — |
| User selects provider (e.g., Anthropic) | Prompts for API key or setup token | — |
| User enters API key | Validates format, writes credential | `~/.openclaw/credentials/` |
| Model picker displayed | Lists available models for provider | — |
| User selects model | Writes `agents.defaults.model` | `openclaw.json` |

**OAuth providers** (GitHub Copilot, Gemini CLI auth): launch browser flow, exchange tokens, store OAuth credentials.

**Custom API:** Prompts for base URL, API key, model ID. Writes to `models.providers` config.

#### Step 3: Gateway Configuration

**Actor:** User → Gateway config prompts

| Setting | QuickStart Default | Advanced Prompt |
|---|---|---|
| Port | `18789` | Prompted |
| Bind mode | `loopback` | `loopback` / `lan` / `tailnet` / `auto` / `custom` |
| Auth mode | `token` (auto-generated) | `token` / `password` |
| Tailscale | `off` | `off` / `serve` / `funnel` |

**Safety constraints:** Tailscale mode forces `bind=loopback`. Funnel mode forces `password` auth.

**Security defaults for new installs:** `gateway.nodes.denyCommands` set to block high-risk commands: `camera.snap`, `camera.clip`, `screen.record`, `calendar.add`, `contacts.add`, `reminders.add`.

**Data updated:** Gateway config section in `openclaw.json`.

#### Step 4: Channel Connection

**Actor:** User → Channel onboarding adapters

Each channel has a specialized setup flow:

**Telegram:**

| Action | System Response | Data Updated |
|---|---|---|
| BotFather instructions displayed | Step-by-step token creation guide | — |
| User enters bot token | Validates token format | `channels.telegram.botToken` |
| AllowFrom prompt (if quickstart) | Prompts for Telegram usernames/IDs | `channels.telegram.allowFrom[]` |

**WhatsApp:**

| Action | System Response | Data Updated |
|---|---|---|
| QR link prompt | "Link via QR now?" | — |
| User confirms | Baileys session starts, QR displayed in terminal | `~/.openclaw/sessions/<account>/auth/creds.json` |
| User scans QR with phone | Connection established | Session credentials persisted |
| Phone type prompt | Personal vs. dedicated number | `channels.whatsapp.dmPolicy`, `allowFrom[]` |

**Discord:**

| Action | System Response | Data Updated |
|---|---|---|
| Developer Portal instructions | Token creation guide + Message Content Intent note | — |
| User enters bot token | Validates token | `channels.discord.token` |
| Guild/channel allowlist | Resolves server/channel names → IDs via Discord API | `channels.discord.guilds{}` |

**Other channels:** Similar pattern — token/credential prompt → validation → config write. Some channels (Matrix, MS Teams) require additional OAuth or app registration steps.

#### Step 5: DM Policy Configuration

**Actor:** User → DM policy prompts

| Action | System Response | Data Updated |
|---|---|---|
| "Configure DM access policies?" | Per-channel policy prompt | — |
| User selects policy per channel | `pairing` / `allowlist` / `open` / `disabled` | `channels.<ch>.dmPolicy` |
| If `allowlist`: prompt sender IDs | Channel-specific ID resolution | `channels.<ch>.allowFrom[]` |

QuickStart mode skips this step (defaults to `pairing` for all channels).

#### Step 6: Skills and Hooks

**Actor:** Wizard → Config store

| Action | System Response | Data Updated |
|---|---|---|
| Skills prompt | Install/configure available agent skills | `skills` config, skill files in workspace |
| Internal hooks prompt | "Clear memory on /new?" | `hooks.internal` config |
| Workspace setup | Creates `~/.openclaw/agents/<agentId>/workspace/` | Directory structure |

#### Step 7: Gateway Start and Verification

**Actor:** Wizard → OS service manager → Gateway process

| Action | System Response | Data Updated |
|---|---|---|
| "Install gateway service?" | Recommended; QuickStart auto-installs | — |
| Runtime prompt | `node` (default) or `bun` | Service config |
| Service install | systemd user service (Linux) or LaunchAgent (macOS) | OS service files |
| Wait for reachable | Polls gateway HTTP for up to 15 seconds | — |
| Health check | `openclaw health` | — |
| Hatch choice | `tui` (recommended) / `web` (open browser) / `later` | — |

**TUI launch:** Opens interactive terminal UI with initial wake message.
**Web launch:** Opens Control UI URL in browser with embedded auth token.

### Error Recovery

| Error | Recovery |
|---|---|
| Invalid API key | Re-prompts for key |
| Channel token validation fails | Shows error, re-prompts |
| WhatsApp QR scan timeout | Offers retry |
| Gateway fails to start | Shows error log, suggests `openclaw doctor` |
| Config file corrupt | Offers reset or manual edit |

---

## 3.4 Cross-Workflow Interactions

### How Workflows Connect

```
                    ┌──────────────────────────┐
                    │       Onboarding         │
                    │  (one-time setup)        │
                    └──────────┬───────────────┘
                               │ produces config
                               ▼
                    ┌──────────────────────────┐
                    │    Gateway Startup        │
                    │  loads config, starts     │
                    │  channels + heartbeat     │
                    └────┬────────────┬────────┘
                         │            │
              ┌──────────▼──┐  ┌──────▼──────────┐
              │  Inbound    │  │   Heartbeat      │
              │  Message    │  │   Runner          │
              │  Routing    │  │                   │
              └──────┬──────┘  └───────┬───────────┘
                     │                 │
                     │  both use:      │
                     ▼                 ▼
              ┌──────────────────────────────┐
              │      Agent Engine            │
              │  (shared LLM call pipeline,  │
              │   tools, session store,      │
              │   memory search)             │
              └──────────┬──────────────────┘
                         │
                         ▼
              ┌──────────────────────────────┐
              │    Outbound Delivery         │
              │  (shared chunking, format,   │
              │   channel adapters)          │
              └──────────────────────────────┘
```

### Shared Components

| Component | Used By | Purpose |
|---|---|---|
| `getReplyFromConfig()` | Inbound routing, Heartbeat, Cron, Hooks | Unified agent turn pipeline |
| `resolveAgentRoute()` | Inbound routing, Exec approvals | Binding → agent + session resolution |
| `deliverOutboundPayloads()` | Inbound routing, Heartbeat, Cron delivery | Unified outbound delivery |
| Session store | All workflows | Shared conversation persistence |
| Command queue lanes | Inbound (`main`), Cron (`cron`), Sub-agents (`subagent`) | Concurrency serialization |
| Memory search | Agent turns (any source) | Context enrichment from vector store |
