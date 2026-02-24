# 02 — Data Model

## 2.1 Database Engine

- **Engine:** SQLite 3 via `better-sqlite3` (synchronous, in-process)
- **Location:** `store/messages.db`
- **ORM:** None — raw SQL with prepared statements
- **Migrations:** Inline `ALTER TABLE` with try/catch for idempotency (no migration framework)
- **Test mode:** In-memory database (`:memory:`)

---

## 2.2 Entity Catalog

### 2.2.1 Chat

Metadata about every conversation the system has seen. Not all chats are registered groups — this table tracks any chat that has sent a message, enabling group discovery.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `jid` | TEXT | **PK** | Unique chat identifier (WhatsApp JID, `tg:` prefix for Telegram, `dc:` for Discord) |
| `name` | TEXT | nullable | Human-readable display name |
| `last_message_time` | TEXT | nullable | ISO 8601 timestamp of most recent message |
| `channel` | TEXT | nullable | Channel type: `whatsapp`, `telegram`, `discord` |
| `is_group` | INTEGER | DEFAULT 0 | Boolean: 1 = group chat, 0 = solo/DM |

**Special row:** JID `__group_sync__` stores the last WhatsApp group metadata sync timestamp.

**Upsert behavior:** On conflict, `name` is updated if provided; `last_message_time` uses `MAX()` to never go backward; `channel` and `is_group` use `COALESCE()` to preserve existing values.

---

### 2.2.2 Message

Full message content for registered groups. Only stored for chats that are registered — unregistered chats only get metadata in the `chats` table.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | **PK** (composite) | Message ID (unique per chat, from channel) |
| `chat_jid` | TEXT | **PK** (composite), **FK → chats.jid** | Owning chat |
| `sender` | TEXT | | Sender identifier (JID or channel-specific ID) |
| `sender_name` | TEXT | | Human-readable sender name |
| `content` | TEXT | | Message body text |
| `timestamp` | TEXT | indexed | ISO 8601 timestamp |
| `is_from_me` | INTEGER | | Boolean: sent by the bot's own number |
| `is_bot_message` | INTEGER | DEFAULT 0 | Boolean: agent-generated response |

**Index:** `idx_timestamp` on `timestamp` — used by the message loop's cursor-based polling.

**Query patterns:**
- `getNewMessages(jids[], lastTimestamp)` — cursor scan across multiple chats, excludes bot messages
- `getMessagesSince(chatJid, sinceTimestamp)` — single-chat history for context accumulation

**Bot message filtering:** Dual filter — checks both `is_bot_message = 0` flag AND `content NOT LIKE '{ASSISTANT_NAME}:%'` for backward compatibility with pre-migration messages.

---

### 2.2.3 RegisteredGroup

Maps a WhatsApp chat to a filesystem folder and container configuration. This is the core "enrollment" table — only registered chats trigger agent processing.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `jid` | TEXT | **PK** | WhatsApp chat JID |
| `name` | TEXT | NOT NULL | Display name |
| `folder` | TEXT | NOT NULL, **UNIQUE** | Directory name under `groups/` (validated: `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`) |
| `trigger_pattern` | TEXT | NOT NULL | Regex pattern that activates the agent (e.g., `^@Andy\b`) |
| `added_at` | TEXT | NOT NULL | ISO 8601 registration timestamp |
| `container_config` | TEXT | nullable | JSON blob: `{ additionalMounts?, timeout? }` |
| `requires_trigger` | INTEGER | DEFAULT 1 | Boolean: 0 = all messages trigger agent (main/DMs), 1 = pattern match required |

**In-memory representation:** Loaded into a `Record<string, RegisteredGroup>` map (keyed by JID) at startup. The TypeScript interface renames columns: `trigger_pattern` → `trigger`, `container_config` → parsed `ContainerConfig` object.

**Folder validation:** On read, every row is checked with `isValidGroupFolder()`. Invalid folders are silently skipped with a warning log. On write, invalid folders throw.

**Reserved folder:** `"global"` cannot be used as a group folder (reserved for shared read-only memory).

---

### 2.2.4 ScheduledTask

Recurring or one-shot jobs that spawn containers on a schedule.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | **PK** | Unique task identifier |
| `group_folder` | TEXT | NOT NULL | Owning group's folder name |
| `chat_jid` | TEXT | NOT NULL | Target chat for result delivery |
| `prompt` | TEXT | NOT NULL | Instructions passed to the agent |
| `schedule_type` | TEXT | NOT NULL | `'cron'` \| `'interval'` \| `'once'` |
| `schedule_value` | TEXT | NOT NULL | Cron expression, milliseconds (as string), or ISO 8601 timestamp |
| `context_mode` | TEXT | DEFAULT `'isolated'` | `'group'` = reuse group session, `'isolated'` = fresh container |
| `next_run` | TEXT | nullable, indexed | ISO 8601 of next scheduled execution |
| `last_run` | TEXT | nullable | ISO 8601 of most recent execution |
| `last_result` | TEXT | nullable | Truncated output from last run |
| `status` | TEXT | DEFAULT `'active'`, indexed | `'active'` \| `'paused'` \| `'completed'` |
| `created_at` | TEXT | NOT NULL | ISO 8601 creation timestamp |

**Indices:** `idx_next_run` (scheduler poll query), `idx_status` (filter active tasks).

**Scheduler query:** `SELECT * FROM scheduled_tasks WHERE status = 'active' AND next_run IS NOT NULL AND next_run <= ? ORDER BY next_run`

**Auto-completion:** When `next_run` is set to `NULL` after a run (only happens for `once` tasks), the `status` is atomically set to `'completed'` in the same UPDATE statement.

---

### 2.2.5 TaskRunLog

Audit trail of every task execution.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | INTEGER | **PK** AUTOINCREMENT | Log entry ID |
| `task_id` | TEXT | NOT NULL, **FK → scheduled_tasks.id** | Parent task |
| `run_at` | TEXT | NOT NULL | ISO 8601 execution start time |
| `duration_ms` | INTEGER | NOT NULL | Wall-clock execution duration |
| `status` | TEXT | NOT NULL | `'success'` \| `'error'` |
| `result` | TEXT | nullable | Agent output (may be truncated) |
| `error` | TEXT | nullable | Error message if `status = 'error'` |

**Index:** `idx_task_run_logs` on `(task_id, run_at)` — for per-task history queries.

**Cascade delete:** When a task is deleted, its run logs are explicitly deleted first (`DELETE FROM task_run_logs WHERE task_id = ?`), not via SQL CASCADE.

---

### 2.2.6 Session

Maps each group to its Claude Code session ID for conversation continuity.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `group_folder` | TEXT | **PK** | Group folder name |
| `session_id` | TEXT | NOT NULL | Claude Code session ID (opaque string) |

**Usage:** Loaded into `Record<string, string>` at startup. Updated after each successful container run. Passed to the container via stdin as `sessionId` to resume prior conversation.

---

### 2.2.7 RouterState

Key-value store for polling cursor state. Only two keys are used.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `key` | TEXT | **PK** | State identifier |
| `value` | TEXT | NOT NULL | State value |

**Known keys:**
| Key | Value Format | Purpose |
|-----|-------------|---------|
| `last_timestamp` | ISO 8601 string | Global message cursor — "process messages newer than this" |
| `last_agent_timestamp` | JSON: `{ "chatJid": "timestamp", ... }` | Per-group cursor — "this group last processed messages up to this point" |

---

## 2.3 Entity-Relationship Diagram

```mermaid
erDiagram
    CHAT {
        text jid PK "WhatsApp JID / channel-prefixed ID"
        text name "Display name"
        text last_message_time "ISO 8601"
        text channel "whatsapp | telegram | discord"
        int is_group "0 = DM, 1 = group"
    }

    MESSAGE {
        text id PK "Composite PK with chat_jid"
        text chat_jid PK_FK "→ Chat.jid"
        text sender "Sender JID"
        text sender_name "Display name"
        text content "Message body"
        text timestamp "ISO 8601 (indexed)"
        int is_from_me "Boolean"
        int is_bot_message "Boolean"
    }

    REGISTERED_GROUP {
        text jid PK "WhatsApp JID"
        text name "Display name"
        text folder UK "Filesystem folder (unique)"
        text trigger_pattern "Activation regex"
        text added_at "ISO 8601"
        text container_config "JSON blob (nullable)"
        int requires_trigger "Boolean, default 1"
    }

    SCHEDULED_TASK {
        text id PK "Task identifier"
        text group_folder "Owning group folder"
        text chat_jid "Target chat for results"
        text prompt "Agent instructions"
        text schedule_type "cron | interval | once"
        text schedule_value "Cron expr / ms / ISO date"
        text context_mode "group | isolated"
        text next_run "ISO 8601 (indexed, nullable)"
        text last_run "ISO 8601 (nullable)"
        text last_result "Truncated output (nullable)"
        text status "active | paused | completed"
        text created_at "ISO 8601"
    }

    TASK_RUN_LOG {
        int id PK "Auto-increment"
        text task_id FK "→ ScheduledTask.id"
        text run_at "ISO 8601"
        int duration_ms "Execution time"
        text status "success | error"
        text result "Output (nullable)"
        text error "Error message (nullable)"
    }

    SESSION {
        text group_folder PK "Group folder name"
        text session_id "Claude Code session ID"
    }

    ROUTER_STATE {
        text key PK "State key"
        text value "State value (may be JSON)"
    }

    CHAT ||--o{ MESSAGE : "has messages"
    REGISTERED_GROUP ||--o{ SCHEDULED_TASK : "owns tasks"
    SCHEDULED_TASK ||--o{ TASK_RUN_LOG : "has run history"
    REGISTERED_GROUP ||--|| SESSION : "has session"
```

**Notes on relationships:**
- `Chat ↔ RegisteredGroup` is an **implicit** relationship (both keyed by JID) with no foreign key constraint. A chat can exist without being registered; a registered group always corresponds to a chat.
- `RegisteredGroup ↔ Session` is also implicit (linked by folder name, not FK).
- `ScheduledTask.group_folder` references `RegisteredGroup.folder` but has no FK constraint in SQLite.
- The only enforced FKs are: `Message.chat_jid → Chat.jid` and `TaskRunLog.task_id → ScheduledTask.id`.

---

## 2.4 Non-Database State

Several key entities live outside SQLite:

### 2.4.1 IPC Files (Ephemeral)

| Location | Format | Lifecycle |
|----------|--------|-----------|
| `data/ipc/{group}/messages/*.json` | `{ chatJid, text }` | Written by container → read & deleted by IPC watcher |
| `data/ipc/{group}/tasks/*.json` | `{ type, ...params }` | Written by container → read & deleted by IPC watcher |
| `data/ipc/{group}/input/*.json` | `{ prompt, ... }` | Written by orchestrator → read by container |
| `data/ipc/errors/*.json` | Failed IPC files | Moved here on processing error, kept for debugging |

### 2.4.2 Agent Memory (Persistent)

| Location | Purpose |
|----------|---------|
| `groups/{folder}/CLAUDE.md` | Per-group persistent memory, read/written by the agent |
| `groups/global/` | Shared memory directory, read-only for non-main groups |
| `data/sessions/{group}/.claude/` | Claude Code session files and settings |

### 2.4.3 Container Logs (Persistent)

| Location | Purpose |
|----------|---------|
| `groups/{folder}/logs/container-{timestamp}.log` | Stdout/stderr from each container run |

### 2.4.4 WhatsApp Auth (Persistent)

| Location | Purpose |
|----------|---------|
| `store/auth/` | Multi-file auth state for Baileys (creds, keys, sessions) |

---

## 2.5 State Machines

### 2.5.1 ScheduledTask Status

```
                  ┌─────────────────────────┐
                  │                         │
                  ▼                         │
            ┌──────────┐   pause_task   ┌────────┐
  create ──►│  active   │──────────────►│ paused │
            └──────────┘                └────────┘
                  │                         │
                  │ run (once,              │ resume_task
                  │ next_run=null)          │
                  ▼                         │
            ┌───────────┐                   │
            │ completed │     ◄─────────────┘
            └───────────┘         (no — resume
                                  returns to active)

  Any state ──► [deleted] via cancel_task
                (hard delete from DB, not a status value)
```

**Transitions:**

| From | To | Trigger | Side Effect |
|------|----|---------|-------------|
| *(new)* | `active` | `createTask()` | `next_run` calculated from schedule |
| `active` | `active` | Successful run (cron/interval) | `next_run` recalculated, `last_run` updated |
| `active` | `completed` | Successful run (once) OR `next_run` set to NULL | Auto-transition in `updateTaskAfterRun()` |
| `active` | `paused` | `pause_task` IPC command | `next_run` preserved for resume |
| `paused` | `active` | `resume_task` IPC command | `next_run` recalculated from current time |
| Any | *(deleted)* | `cancel_task` IPC command | Row deleted + cascade delete of `task_run_logs` |

### 2.5.2 TaskRunLog Status

Simple binary — no transitions, written once per execution:

```
  run starts ──► success   (result populated)
             └─► error     (error populated)
```

### 2.5.3 Container Lifecycle (In-Memory, Not Persisted)

The `GroupQueue` tracks container state per group in memory. This is not stored in the database.

```
                           new messages/tasks
                                  │
                                  ▼
  ┌─────────┐  slot available  ┌────────┐  agent completes  ┌──────────────┐
  │ queued   │────────────────►│ active  │─────────────────►│ idle_waiting  │
  └─────────┘                  └────────┘                   └──────────────┘
       ▲                          │                              │    │
       │                          │ error                        │    │ 30min
       │                          ▼                              │    │ timeout
       │                    ┌───────────┐   retry delay          │    │
       │                    │  retrying  │──────────┘             │    │
       │                    └───────────┘                         │    ▼
       │                          │                              │  ┌──────┐
       │                          │ max retries                  │  │ done │
       │                          ▼                              │  └──────┘
       │                    ┌──────────┐                         │
       └────────────────────│ dropped  │                         │
          (next msg requeues)└──────────┘                         │
                                                                 │
                                              new messages ──────┘
                                              piped via IPC input
```

**Key transitions:**
- `queued → active`: A concurrency slot opens (`activeCount < MAX_CONCURRENT_CONTAINERS`)
- `active → idle_waiting`: Container finished processing but stays alive for follow-ups
- `idle_waiting → active`: New messages piped to running container via IPC input files
- `idle_waiting → done`: 30-minute idle timeout, container killed
- `active → retrying`: Processing error, exponential backoff (5s × 2^n)
- `retrying → queued`: After delay, re-enters queue
- `retrying → dropped`: Max 5 retries exceeded, messages discarded (next incoming message re-triggers)

---

## 2.6 Data Flow Summary

```
  WhatsApp ──► storeMessage() ──► messages table
                                      │
                   2s poll ◄───────────┘
                      │
                      ▼
              getNewMessages(jids, cursor)
                      │
                      ▼
              [trigger check]
                      │
                      ▼
              getMessagesSince(jid, agentCursor)
                      │
                      ▼
              Container (stdin JSON)
                      │
                      ▼
              IPC output file
                      │
                      ▼
              sendMessage(jid, text) ──► WhatsApp
                      │
                      ▼
              setSession(), setRouterState()
```

## 2.7 Migration Strategy

The codebase uses **inline idempotent migrations** — `ALTER TABLE ... ADD COLUMN` wrapped in try/catch. If the column already exists, the error is silently caught. This runs on every startup.

Additionally, there is a one-time **JSON-to-SQLite migration** (`migrateJsonState()`) that imports data from legacy JSON files (`router_state.json`, `sessions.json`, `registered_groups.json`), then renames them to `.migrated`.

There is no migration versioning table or framework. New schema changes are added as new try/catch blocks in `createSchema()`.
