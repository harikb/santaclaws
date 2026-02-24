# 03 — Core Workflows

This document traces the three primary workflows end-to-end: the actors involved, each action taken, the system's response, and which data is updated at every step.

---

## 3.1 Message Ingestion → Agent Response

The most complex workflow. A user sends a WhatsApp message; the system detects it, decides whether to invoke the AI agent, spawns an isolated container, streams the response back, and updates cursors for crash recovery.

### Flow Diagram

```
User (WhatsApp)
  │
  ▼
WhatsApp Socket (Baileys 'messages.upsert')
  │
  ├──► storeChatMetadata()          → chats table
  ├──► storeMessage()               → messages table
  │
  ▼
Message Loop (2s poll)
  │
  ├──► getNewMessages(jids, cursor) → SQL cursor scan
  ├──► Advance lastTimestamp        → router_state table
  │
  ▼
Per-Group Processing
  │
  ├──► Trigger check (non-main groups only)
  ├──► getMessagesSince(jid, agentCursor) → accumulated context
  ├──► formatMessages() → XML prompt
  ├──► Advance lastAgentTimestamp   → router_state table
  │
  ▼
GroupQueue
  │
  ├─ Active container? → Pipe via IPC input file
  ├─ Slot available?   → Spawn new container
  └─ At capacity?      → Queue (waiting list)
  │
  ▼
Container Execution
  │
  ├──► stdin: JSON { prompt, sessionId, secrets }
  ├──► stdout: ---OUTPUT_START--- {result} ---OUTPUT_END---
  │
  ▼
Streaming Callback
  │
  ├──► Strip <internal> tags
  ├──► channel.sendMessage(jid, text) → WhatsApp
  ├──► Persist newSessionId          → sessions table
  └──► Start idle timer (30 min)
  │
  ▼
Cleanup
  │
  ├──► Write container log           → groups/{folder}/logs/
  ├──► Success: cursor committed
  └──► Error + no output: rollback cursor, retry with backoff
```

### Step-by-Step

#### Step 1 — WhatsApp Message Arrival

| | |
|---|---|
| **Actor** | WhatsApp user sends a message in a group or DM |
| **Action** | Baileys socket emits `messages.upsert` event |
| **System Response** | `WhatsAppChannel` callback fires (`whatsapp.ts:149-203`). Parses message: extracts JID, timestamp (epoch → ISO 8601), sender, content. Detects bot messages via `is_from_me` flag or content prefix pattern. |
| **Data Updated** | `storeChatMetadata()` → upserts `chats` row (JID, name, timestamp, channel, is_group). `storeMessage()` → inserts `messages` row (only for registered groups). |
| **Gate** | If the chat is not in `registeredGroups`, only metadata is stored — no message content is persisted and no further processing occurs. |

#### Step 2 — Message Loop Polling

| | |
|---|---|
| **Actor** | `startMessageLoop()` — async loop running every `POLL_INTERVAL` (2,000 ms) |
| **Action** | Calls `getNewMessages(jids, lastTimestamp, ASSISTANT_NAME)` |
| **System Response** | SQL query: `SELECT FROM messages WHERE timestamp > ? AND chat_jid IN (?) AND is_bot_message = 0 AND content NOT LIKE '{name}:%'`. Returns all non-bot messages across registered groups since the last cursor position. |
| **Data Updated** | `lastTimestamp` advanced to the max timestamp from results. Saved immediately via `setRouterState('last_timestamp', ...)`. This cursor advance happens **before** processing — it's a deliberate design choice for crash recovery (see Step 7). |
| **Gate** | If no new messages: sleep and re-poll. If messages exist: group by `chat_jid` and proceed per-group. |

#### Step 3 — Trigger Detection

| | |
|---|---|
| **Actor** | Message loop, per-group |
| **Action** | Check whether the group requires a trigger pattern match |
| **System Response** | **Main group** (`folder === "main"`): always processes, no trigger needed (`requiresTrigger` defaults to `false`). **Non-main groups**: scans all new messages for `TRIGGER_PATTERN` match (default: `/^@Andy\b/i`). |
| **Data Updated** | None — this is a read-only check. |
| **Gate** | If no trigger found in non-main group: **skip entirely**. Messages remain in the database and will be included as **accumulated context** when a future trigger arrives (Step 4). |

#### Step 4 — Context Accumulation & Formatting

| | |
|---|---|
| **Actor** | `processGroupMessages()` (`index.ts:131-227`) |
| **Action** | Pulls all messages since the group's last agent processing, not just the trigger batch |
| **System Response** | `getMessagesSince(chatJid, lastAgentTimestamp[chatJid])` — retrieves every non-bot message since the agent last saw this group. This includes messages from previous polling cycles that didn't have a trigger. `formatMessages()` (`router.ts`) wraps them in XML: `<messages><message sender="Name" time="ISO">Content</message>...</messages>` |
| **Data Updated** | `lastAgentTimestamp[chatJid]` advanced to the latest message timestamp. Saved via `setRouterState('last_agent_timestamp', JSON.stringify({...}))`. Cursor saved **before** container execution — see crash recovery in Step 7. |

#### Step 5 — Queue & Container Decision

| | |
|---|---|
| **Actor** | `GroupQueue` (`group-queue.ts`) |
| **Action** | Determine whether to pipe to an existing container or spawn a new one |
| **System Response** | Three possible paths: **(a)** Container already active and idle → write formatted message to `data/ipc/{group}/input/{timestamp}.json` (atomic write: `.tmp` → rename). Container reads it and continues. **(b)** Slot available (`activeCount < MAX_CONCURRENT_CONTAINERS`) → spawn new container via `runForGroup()`. **(c)** At capacity → add to `waitingGroups` queue, set `pendingMessages = true`. |
| **Data Updated** | `activeCount` incremented (if spawning). Group state: `active = true`, `isTaskContainer = false`. |
| **Priority** | Pending tasks always drain before pending messages within the same group. |

#### Step 6 — Container Spawning & Agent Execution

| | |
|---|---|
| **Actor** | `runContainerAgent()` (`container-runner.ts:218-585`) |
| **Action** | Build volume mounts, spawn Docker/Apple Container, pass input via stdin |
| **System Response** | `buildVolumeMounts()` creates the isolation profile (see §3.1.1 below). Container spawned with `docker run -i --rm --name nanoclaw-{folder}-{timestamp}`. Input JSON written to stdin including `{ prompt, sessionId, groupFolder, chatJid, isMain, secrets }`. Secrets deleted from object immediately after write. stdin closed. |
| **Data Updated** | Container process registered with GroupQueue (`state.process`, `state.containerName`). |
| **Timeout** | Hard timeout = `max(CONTAINER_TIMEOUT, IDLE_TIMEOUT + 30s)`. Resets on each streaming output marker. |

##### 3.1.1 Container Mount Profiles

**Main group:**
| Mount Point | Host Path | Mode |
|-------------|-----------|------|
| `/workspace/project` | Project root | Read-only |
| `/workspace/group` | `groups/main/` | Read-write |
| `/workspace/ipc` | `data/ipc/main/` | Read-write |
| `/home/node/.claude` | `data/sessions/main/.claude/` | Read-write |
| `/app/src` | Per-group agent-runner copy | Read-write |

**Regular group:**
| Mount Point | Host Path | Mode |
|-------------|-----------|------|
| `/workspace/group` | `groups/{folder}/` | Read-write |
| `/workspace/global` | `groups/global/` (if exists) | Read-only |
| `/workspace/ipc` | `data/ipc/{folder}/` | Read-write |
| `/home/node/.claude` | `data/sessions/{folder}/.claude/` | Read-write |
| `/app/src` | Per-group agent-runner copy | Read-write |
| `/workspace/extra/*` | Additional validated mounts | Per-allowlist |

#### Step 7 — Streaming Output & Delivery

| | |
|---|---|
| **Actor** | Container stdout stream parser |
| **Action** | Parse `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---` marker pairs from stdout |
| **System Response** | For each marker pair: extract JSON (`{ status, result, newSessionId?, error? }`). Strip `<internal>...</internal>` blocks (agent reasoning scratchpad). If result text is non-empty, call `channel.sendMessage(chatJid, text)` — delivered to WhatsApp in real-time. If `newSessionId` present, persist via `setSession()`. Start/reset idle timer (30 min). |
| **Data Updated** | `sessions` table (if new session ID). `outputSentToUser` flag set (affects crash recovery). |
| **Delivery** | If `ASSISTANT_HAS_OWN_NUMBER` is false, message is prefixed with `{ASSISTANT_NAME}: `. If WhatsApp is disconnected, message queued in `outgoingQueue[]` and sent on reconnect. |

#### Step 8 — Cleanup & Crash Recovery

| | |
|---|---|
| **Actor** | Container `close` event handler |
| **Action** | Container exits (normally, timeout, or error) |
| **System Response** | Write detailed log to `groups/{folder}/logs/container-{timestamp}.log` (input summary, stderr, stdout, exit code). Timeout with prior output = success (idle cleanup). Timeout without output = error. |
| **Data Updated** | `activeCount` decremented. Group state reset. Queue drains: pending tasks first, then messages, then waiting groups. |

**Crash recovery logic (cursor management):**

| Scenario | Cursor Behavior |
|----------|----------------|
| Success | `lastAgentTimestamp` stays advanced (committed before execution) |
| Error, but output was sent to user | `lastAgentTimestamp` stays advanced (prevents duplicate messages) |
| Error, no output sent | `lastAgentTimestamp` **rolled back** to previous value (messages re-processed on retry) |
| Process crash between cursor advance and execution | `recoverPendingMessages()` on startup re-scans all groups for unprocessed messages |

**Retry behavior:** Exponential backoff — 5s, 10s, 20s, 40s, 80s. Max 5 retries. After exhaustion, messages are dropped but will re-trigger on next incoming message.

---

## 3.2 Scheduled Task Lifecycle

Tasks are created by agents (via IPC), polled by the scheduler, executed in containers, and automatically rescheduled.

### Flow Diagram

```
Agent (in container)
  │
  ▼
IPC file: /data/ipc/{group}/tasks/schedule_task.json
  │
  ▼
IPC Watcher (1s poll)
  │
  ├──► Authorization check (main can target any group)
  ├──► Calculate next_run from schedule_type
  └──► createTask()                  → scheduled_tasks table
  │
  ▼
Scheduler Loop (60s poll)
  │
  ├──► getDueTasks() WHERE status='active' AND next_run <= NOW()
  ├──► Re-verify status (may have been paused)
  └──► queue.enqueueTask()
  │
  ▼
GroupQueue (same as messages)
  │
  ▼
runTask()
  │
  ├──► Container spawned (context_mode determines session reuse)
  ├──► Output streamed → sendMessage() to target chat
  ├──► 10s grace period → container closed
  │
  ▼
Post-Execution
  │
  ├──► logTaskRun()                  → task_run_logs table
  ├──► Calculate next_run (cron/interval) or NULL (once)
  └──► updateTaskAfterRun()          → scheduled_tasks table
```

### Step-by-Step

#### Step 1 — Task Creation (via IPC)

| | |
|---|---|
| **Actor** | Agent running inside a container |
| **Action** | Writes JSON file to `/workspace/ipc/tasks/{filename}.json` with `type: "schedule_task"` |
| **System Response** | IPC watcher reads file, calls `processTaskIpc()`. Authorization: main group can create tasks for any group; non-main groups can only create for themselves. |
| **Data Updated** | `createTask()` → INSERT into `scheduled_tasks` with `status: 'active'`, calculated `next_run`. |

**`next_run` calculation by schedule type:**

| Type | Input Example | Calculation |
|------|--------------|-------------|
| `cron` | `"0 9 * * *"` | `CronExpressionParser.parse(value, { tz: TIMEZONE }).next().toISOString()` |
| `interval` | `"86400000"` (24h in ms) | `new Date(Date.now() + parseInt(value)).toISOString()` |
| `once` | `"2026-02-23T10:00:00Z"` | `new Date(value).toISOString()` |

#### Step 2 — Scheduler Polling

| | |
|---|---|
| **Actor** | `startSchedulerLoop()` — polls every `SCHEDULER_POLL_INTERVAL` (60,000 ms) |
| **Action** | `getDueTasks()` — query `scheduled_tasks WHERE status = 'active' AND next_run <= NOW()` |
| **System Response** | For each due task: re-fetch from DB to verify status hasn't changed (pause/cancel may have occurred between poll and execution). If still active, enqueue via `queue.enqueueTask()`. |
| **Data Updated** | None at this step — queuing is in-memory only. |

#### Step 3 — Task Execution

| | |
|---|---|
| **Actor** | `runTask()` (`task-scheduler.ts:34-204`) |
| **Action** | Spawn container with task prompt. `context_mode` determines session: `'group'` reuses the group's persistent Claude session; `'isolated'` (default) starts fresh. |
| **System Response** | Same container lifecycle as message processing. Key difference: `isScheduledTask: true` flag. After first output marker received, a 10-second close delay is scheduled (`TASK_CLOSE_DELAY_MS`) — tasks are single-turn, so the container is proactively closed after delivering its result. |
| **Data Updated** | Output sent to `chat_jid` via `channel.sendMessage()`. |

#### Step 4 — Post-Execution & Rescheduling

| | |
|---|---|
| **Actor** | `runTask()` cleanup block |
| **Action** | Log execution, calculate next run, update task record |
| **System Response** | `logTaskRun()` inserts audit row. `updateTaskAfterRun()` atomically updates `next_run`, `last_run`, `last_result`, and conditionally sets `status = 'completed'` if `next_run` is NULL (only for `once` tasks). |
| **Data Updated** | `task_run_logs`: new row (task_id, run_at, duration_ms, status, result/error). `scheduled_tasks`: next_run recalculated, last_run/last_result updated. |

**Rescheduling logic:**

| Type | After Execution |
|------|----------------|
| `cron` | `next_run` = next cron occurrence from current time |
| `interval` | `next_run` = `now + interval_ms` |
| `once` | `next_run` = NULL → status auto-set to `'completed'` |

#### Step 5 — Task Control Operations

Agents can manage tasks through IPC control files:

| IPC Type | Authorization | Effect |
|----------|--------------|--------|
| `pause_task` | Own group or main | `status → 'paused'` (scheduler skips) |
| `resume_task` | Own group or main | `status → 'active'` (scheduler resumes, uses existing `next_run`) |
| `cancel_task` | Own group or main | Hard delete: row removed from `scheduled_tasks` + cascade delete from `task_run_logs` |

---

## 3.3 Group Registration

New groups are onboarded through an IPC request from the main group's agent. This is the only way to add groups — there is no admin UI or CLI command.

### Flow Diagram

```
Agent (main group container)
  │
  ▼
IPC file: /data/ipc/main/tasks/register_group.json
  │
  ▼
IPC Watcher
  │
  ├──► Authorization: isMain? (REJECT if not)
  ├──► Validate folder name (regex, reserved names, traversal)
  │
  ▼
registerGroup()
  │
  ├──► registeredGroups[jid] = group   (in-memory, immediate effect)
  ├──► setRegisteredGroup()            → registered_groups table
  └──► mkdirSync(groups/{folder}/logs) → filesystem
  │
  ▼
Message Loop (next poll)
  │
  └──► New group's JID included in getNewMessages() query
       └──► Messages now flow through standard pipeline
```

### Step-by-Step

#### Step 1 — Registration Request

| | |
|---|---|
| **Actor** | Agent running in the main group's container |
| **Action** | Writes JSON to `/workspace/ipc/tasks/{filename}.json` with `type: "register_group"` |
| **Payload** | `{ jid, name, folder, trigger, requiresTrigger?, containerConfig? }` |
| **Authorization** | **Main group only.** Determined by IPC directory path (`/data/ipc/main/`). Non-main groups cannot register new groups — prevents privilege escalation. |

#### Step 2 — Validation

| | |
|---|---|
| **Actor** | IPC watcher (`ipc.ts:351-382`) |
| **Action** | Validate folder name against security rules |
| **System Response** | `isValidGroupFolder()` checks: regex `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`, no whitespace, no path separators (`/`, `\`), no traversal (`..`), not in reserved set (`global`). Rejects invalid names with warning log. |
| **Gate** | Invalid folder → reject silently. Missing required fields → reject silently. |

#### Step 3 — Persistence & Activation

| | |
|---|---|
| **Actor** | `registerGroup()` (`index.ts:80-102`) |
| **Action** | Update in-memory state, persist to DB, create filesystem structure |
| **System Response** | `registeredGroups[jid] = group` — immediate visibility to message loop. `setRegisteredGroup()` — INSERT OR REPLACE into `registered_groups` table. `mkdirSync()` — create `groups/{folder}/` and `groups/{folder}/logs/`. |
| **Data Updated** | `registered_groups` table (jid, name, folder, trigger_pattern, added_at, container_config, requires_trigger). Filesystem: group directory + logs subdirectory. |

#### Step 4 — Activation Effects

| | |
|---|---|
| **Actor** | Message loop (next poll cycle) |
| **Action** | New JID included in `Object.keys(registeredGroups)` |
| **System Response** | `getNewMessages()` now queries this JID. Future messages are stored with full content. Trigger detection applies per the group's `trigger_pattern` and `requiresTrigger` setting. When triggered, a container is spawned with the group's isolation profile: own folder (RW), global (RO), no project access, isolated `.claude/` session, isolated IPC namespace. |
| **Timing** | Activation is effectively instant — the in-memory update takes effect on the next 2-second poll cycle. |

---

## 3.4 Supporting Workflows

### 3.4.1 Crash Recovery

On every startup, `recoverPendingMessages()` (`index.ts:404-416`) scans all registered groups:

1. For each group, compare `lastAgentTimestamp[chatJid]` against unprocessed messages in the `messages` table.
2. If unprocessed messages exist → `queue.enqueueMessageCheck(chatJid)`.
3. This handles the gap between cursor advancement (Step 4 in §3.1) and successful processing.

### 3.4.2 Idle Container Reuse

When a container finishes processing but hasn't timed out:

1. Container marked `idleWaiting = true` via `queue.notifyIdle()`.
2. If new messages arrive for the same group within `IDLE_TIMEOUT` (30 min):
   - Formatted message written to `data/ipc/{group}/input/{timestamp}.json`.
   - Container reads the file and processes without restart.
   - No new container spawn needed.
3. If idle timeout expires: `queue.closeStdin()` → container exits gracefully.

### 3.4.3 WhatsApp Reconnection

When WhatsApp disconnects and reconnects (`whatsapp.ts`):

1. `outgoingQueue[]` is flushed — any messages queued during disconnect are sent.
2. Message loop continues polling from `lastTimestamp` — no messages are lost (they're already in SQLite).
3. Group metadata sync may be triggered if stale (24-hour cache).

### 3.4.4 Group Metadata Sync

Main group can request a refresh via IPC (`refresh_groups`):

1. Calls `syncGroupMetadata(force=true)` on the WhatsApp channel.
2. Fetches all group metadata from WhatsApp.
3. Updates `chats` table with current group names.
4. Records sync timestamp in special `__group_sync__` row.
5. Cached for 24 hours (`GROUP_SYNC_INTERVAL_MS`).
