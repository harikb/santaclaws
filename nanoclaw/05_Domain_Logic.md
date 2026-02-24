# 05 — Domain Logic & Technical Debt

This document catalogs the business rules enforced in code, validation constraints, edge cases, and areas of technical debt that a rebuild team should be aware of.

---

## 5.1 Business Rules

### 5.1.1 Trigger & Message Routing Rules

| Rule | Enforcement Location | Description |
|------|---------------------|-------------|
| **Main group never requires trigger** | `index.ts:345-349` | The group with `folder === "main"` processes all messages. `requiresTrigger` defaults to `false`. |
| **Non-main groups require trigger** | `index.ts:351-362` | Messages must match `TRIGGER_PATTERN` (default: `/^@Andy\b/i`). Non-trigger messages silently accumulate as context. |
| **Context accumulation** | `index.ts:366-370` | When a trigger fires, `getMessagesSince(lastAgentTimestamp)` pulls ALL non-bot messages since the agent last processed this group — including messages from prior polling cycles that had no trigger. |
| **Bot message exclusion** | `db.ts:293-298` | Bot messages filtered by two conditions: `is_bot_message = 0` AND `content NOT LIKE '{name}:%'`. The dual filter exists for backward compatibility with pre-migration messages. |
| **Unregistered chats ignored** | `whatsapp.ts:167-168` | Only metadata (JID, name, timestamp) is stored for unregistered chats. Message content is discarded. |
| **`<internal>` tag stripping** | `router.ts` | Agent output wrapped in `<internal>...</internal>` is stripped before delivery to the user. Agents use this for reasoning/scratchpad. |

### 5.1.2 Container & Execution Rules

| Rule | Enforcement Location | Description |
|------|---------------------|-------------|
| **Max 5 concurrent containers** | `group-queue.ts:71,108` | Global limit (`MAX_CONCURRENT_CONTAINERS`). Excess work queued with per-group ordering. |
| **Tasks drain before messages** | `group-queue.ts:277-284` | Within a group, pending scheduled tasks are processed before pending messages. |
| **Idle containers reused** | `group-queue.ts:137-143` | After completing work, containers stay alive for `IDLE_TIMEOUT` (30 min). Follow-up messages piped via IPC input files. |
| **Tasks are single-turn** | `task-scheduler.ts:116` | After a task produces output, the container is closed after a 10-second grace period (`TASK_CLOSE_DELAY_MS`). |
| **Session reuse per context mode** | `task-scheduler.ts:108-111` | `context_mode: 'group'` reuses the group's Claude session. `context_mode: 'isolated'` starts fresh. |

### 5.1.3 Authorization Rules

| Rule | Enforcement Location | Description |
|------|---------------------|-------------|
| **IPC identity from directory** | `ipc.ts:61` | Source group determined by filesystem path (`/data/ipc/{group}/`), not by any claim in the payload. Cannot be spoofed from inside a container. |
| **Main-only operations** | `ipc.ts:329,353` | `register_group` and `refresh_groups` restricted to `sourceGroup === "main"`. |
| **Cross-group messaging blocked** | `ipc.ts:76-92` | Non-main groups can only send to chats where `registeredGroups[chatJid].folder === sourceGroup`. |
| **Cross-group task management blocked** | `ipc.ts:203,276,294,312` | Non-main groups can only manage tasks where `task.group_folder === sourceGroup`. |
| **Secrets never on disk** | `container-runner.ts:273-277` | Secrets written to container stdin, deleted from input object immediately. Never in env vars, files, or logs. |
| **Bash secrets sanitization** | Agent runner PreToolUse hook | Every Bash command is prepended with `unset ANTHROPIC_API_KEY CLAUDE_CODE_OAUTH_TOKEN` to prevent leakage to child processes. |
| **Mount allowlist external** | `mount-security.ts` | Stored at `~/.config/nanoclaw/mount-allowlist.json`, outside the project root, never mounted into containers. Tamper-proof from agents. |
| **Non-main mounts forced read-only** | `mount-security.ts:297-299` | If `nonMainReadOnly: true` in allowlist, non-main groups can only mount directories as read-only regardless of their configuration. |

### 5.1.4 Naming & Validation Rules

| Rule | Enforcement Location | Description |
|------|---------------------|-------------|
| **Group folder naming** | `group-folder.ts:5` | Must match `/^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$/`. Max 64 chars, alphanumeric start, underscores and hyphens allowed. |
| **Reserved folder name** | `group-folder.ts:6` | `"global"` is reserved (used for shared read-only memory). |
| **Path traversal prevention** | `group-folder.ts:12-14` | Rejects folders containing `..`, `/`, or `\`. |
| **Path escape detection** | `group-folder.ts:24-29` | `ensureWithinBase()` uses `path.relative()` to verify resolved paths don't escape their base directory. |
| **Blocked mount patterns** | `mount-security.ts:29-47` | 17 hardcoded patterns: `.ssh`, `.gnupg`, `.gpg`, `.aws`, `.azure`, `.gcloud`, `.kube`, `.docker`, `credentials`, `.env`, `.netrc`, `.npmrc`, `.pypirc`, `id_rsa`, `id_ed25519`, `private_key`, `.secret`. |

---

## 5.2 Cursor Management & Crash Recovery

The message processing pipeline uses a deliberate "advance-before-process" pattern with specific tradeoffs.

### The Pattern

```
1. Query new messages (cursor = lastTimestamp)
2. Advance lastTimestamp to max(results)     ← saved to DB immediately
3. For each group:
   a. Advance lastAgentTimestamp[group]      ← saved to DB immediately
   b. Spawn container / pipe to active one
   c. Container processes messages
   d. On success: cursor stays advanced
   e. On error (no output): rollback lastAgentTimestamp to previous value
```

### Why Advance Before Process

If the process crashes between step 2 and step 3, messages are not re-fetched by the global cursor. However, `recoverPendingMessages()` on startup re-scans each group's `lastAgentTimestamp` independently, catching any unprocessed messages.

### Recovery Matrix

| Crash Point | Recovery Mechanism | Data Loss Risk |
|-------------|-------------------|----------------|
| After global cursor advance, before group processing | `recoverPendingMessages()` scans per-group cursors | None — per-group cursors haven't advanced |
| After per-group cursor advance, before container spawn | Per-group cursor advanced, but messages still in DB; recovery finds nothing | **Messages lost** — gap between advance and processing |
| During container execution (no output sent) | Cursor rolled back to previous value | None — messages re-processed on retry |
| During container execution (partial output sent) | Cursor NOT rolled back (prevents duplicates) | **Context partially lost** — some messages consumed, error on remainder |
| Container completes successfully | Cursor committed | None |

### Key Constraint

The system prioritizes **no duplicate messages to users** over **no lost context**. If an agent partially responded before failing, the cursor is not rolled back because re-processing would send duplicate responses.

---

## 5.3 Edge Cases

### 5.3.1 Output Marker Collision

The streaming protocol uses text markers (`---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---`) to delimit JSON results in stdout. If an agent's response text happens to contain these exact marker strings, the parser would produce a false split.

**Impact:** Low probability (markers are unusual strings), but would corrupt output parsing.
**Mitigation:** None currently. A binary length-prefix protocol would eliminate this risk.

### 5.3.2 Unbounded Message Queue

When WhatsApp disconnects, outgoing messages are queued in an in-memory array (`outgoingQueue[]`) with no size limit. If the server is unreachable for an extended period while agents continue producing output, this queue could grow unboundedly.

**Impact:** Memory exhaustion on prolonged disconnection.
**Mitigation:** None currently. A max queue size with oldest-message eviction would bound memory use.

### 5.3.3 Silent Message Drop After Max Retries

When message processing fails 5 consecutive times (exponential backoff: 5s → 10s → 20s → 40s → 80s), the retry counter resets and messages are effectively dropped. No error message is sent to the user.

**Impact:** User sends a message, agent never responds, no indication of failure.
**Mitigation:** Could send an error message to the chat after max retries exceeded.

### 5.3.4 Missed Scheduled Tasks

The scheduler polls every 60 seconds. If a task's `next_run` was due during a period when the process was down, `getDueTasks()` returns it on the next poll (since `next_run <= NOW()`). However:

- For `cron` tasks: Next run is calculated from current time, not from the missed run. If the process was down for 3 days, only one catch-up run occurs — the other 2 days' runs are skipped.
- For `interval` tasks: Same behavior — next run = now + interval, not catchup from missed.
- For `once` tasks: The overdue task runs once, then completes. No issue.

### 5.3.5 Bot Message Prefix Collision

In shared-number mode (`ASSISTANT_HAS_OWN_NUMBER = false`), bot messages are detected by checking if content starts with `{ASSISTANT_NAME}:`. If a user's message happens to start with `Andy:`, it would be incorrectly filtered as a bot message.

**Impact:** User message silently ignored.
**Mitigation:** The `is_bot_message` flag (added in a later migration) provides a reliable secondary check, but the content prefix filter is kept as a backstop.

### 5.3.6 Allowlist Cache Permanence

The mount allowlist file is loaded once and cached for the process lifetime. If the file is temporarily unreadable (permissions, disk error), the error is cached permanently — mount validation is broken for the rest of the session.

**Impact:** No additional mounts can be validated until process restart.
**Mitigation:** Could retry allowlist loading on each validation call, or cache with a TTL.

---

## 5.4 Hardcoded Values Reference

### Timing Constants

| Value | Location | Purpose |
|-------|----------|---------|
| `2,000 ms` | `config.ts` | Message loop poll interval |
| `60,000 ms` | `config.ts` | Scheduler poll interval |
| `1,000 ms` | `config.ts` | IPC watcher poll interval |
| `1,800,000 ms` (30 min) | `config.ts` | Container hard timeout |
| `1,800,000 ms` (30 min) | `config.ts` | Idle container timeout |
| `10,000 ms` | `task-scheduler.ts:116` | Task close delay after result |
| `5,000 ms` | `group-queue.ts:15` | Base retry backoff |
| `5,000 ms` | `whatsapp.ts:95` | Reconnection retry delay |
| `86,400,000 ms` (24 h) | `whatsapp.ts:22` | Group metadata sync interval |

### Capacity Limits

| Value | Location | Purpose |
|-------|----------|---------|
| `5` | `config.ts` | Max concurrent containers |
| `5` | `group-queue.ts:14` | Max message processing retries |
| `10,485,760` (10 MB) | `config.ts` | Max stdout/stderr per container |
| `64` chars | `group-folder.ts:5` | Max group folder name length |

### Security Patterns

| Value | Location | Purpose |
|-------|----------|---------|
| 17 blocked patterns | `mount-security.ts:29-47` | Credential directory/file blocklist |
| `global` | `group-folder.ts:6` | Reserved folder name |
| `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$` | `group-folder.ts:5` | Folder name validation regex |

### Protocol Markers

| Value | Location | Purpose |
|-------|----------|---------|
| `---NANOCLAW_OUTPUT_START---` | `container-runner.ts:26` | Stdout result delimiter (open) |
| `---NANOCLAW_OUTPUT_END---` | `container-runner.ts:27` | Stdout result delimiter (close) |
| `_close` | Agent runner | Stdin close sentinel filename |
| `__group_sync__` | `db.ts:222` | Special JID for sync timestamp |

---

## 5.5 Technical Debt

### 5.5.1 No Migration Framework

**Current state:** Schema migrations are inline `ALTER TABLE ... ADD COLUMN` wrapped in try/catch. Each migration runs on every startup and relies on the error being "column already exists."

**Risk:** No versioning, no rollback, no visibility into which migrations have run. Adding a migration that fails partway through leaves the database in an inconsistent state.

**Recommendation:** Adopt a migration table with version tracking (even a simple `schema_version` integer).

### 5.5.2 Polling Instead of Events

**Current state:** Three independent polling loops (2s messages, 60s scheduler, 1s IPC) with fixed intervals.

**Risk:** Adds latency (up to interval duration), wastes CPU cycles on idle polls, and creates subtle timing dependencies. The IPC watcher polls every 1 second even when no containers are running.

**Recommendation:** For IPC, `fs.watch()` / `inotify` would be more efficient. For the scheduler, a single `setTimeout` to the next due task would eliminate polling. Message polling is inherent to the Baileys architecture.

### 5.5.3 In-Memory State Not Durable

**Current state:** `registeredGroups`, `sessions`, `lastAgentTimestamp`, and `lastTimestamp` are loaded from SQLite at startup and maintained in memory. Writes to SQLite happen at specific checkpoints.

**Risk:** If the process crashes between an in-memory update and the corresponding SQLite write, state is lost. The "advance-before-process" pattern mitigates this for cursors, but session IDs and group registrations have no such protection.

**Recommendation:** Write-through to SQLite on every state change (already mostly done, but verify all paths).

### 5.5.4 No Structured Error Reporting to Users

**Current state:** When processing fails (container crash, timeout, max retries), errors are logged to pino but no message is sent to the user's chat. The user sees silence.

**Risk:** Users don't know if the agent is thinking, crashed, or ignored their message.

**Recommendation:** Send a structured error message to the chat on failure (e.g., "Sorry, I encountered an error processing your message. Please try again.").

### 5.5.5 Container Output Truncation Can Break Parsing

**Current state:** Stdout and stderr are independently truncated at `CONTAINER_MAX_OUTPUT_SIZE` (10 MB). If an output marker pair spans the truncation boundary, the marker parser silently fails to find the closing marker.

**Risk:** Agent produces a valid response that's never delivered because truncation split the markers.

**Recommendation:** Truncate only after marker parsing, or use a protocol that doesn't rely on text delimiters (e.g., length-prefixed binary frames).

### 5.5.6 No Database Connection Pooling or WAL Mode

**Current state:** Single `better-sqlite3` instance, synchronous API, no explicit WAL mode configuration.

**Risk:** Synchronous SQLite calls block the Node.js event loop during writes. Without WAL mode, concurrent reads and writes can contend.

**Recommendation:** Enable WAL mode (`PRAGMA journal_mode=WAL`) for better concurrency. Consider whether synchronous writes are acceptable under load.

### 5.5.7 IPC Error Directory Grows Without Bound

**Current state:** Failed IPC files are moved to `data/ipc/errors/` and never cleaned up.

**Risk:** Disk space exhaustion over time if errors are frequent.

**Recommendation:** Add a retention policy (e.g., delete error files older than 7 days on startup).

### 5.5.8 WhatsApp Reconnection Has No Backoff

**Current state:** On disconnection, reconnection is attempted after a fixed 5-second delay. If reconnection fails, there's a single retry and then silence.

**Risk:** Aggressive reconnection during WhatsApp outages. No recovery from persistent failures without process restart.

**Recommendation:** Exponential backoff with jitter, plus a maximum retry count that triggers a user-visible alert.

### 5.5.9 Dual Bot-Message Detection

**Current state:** Bot messages are filtered by both `is_bot_message = 0` flag AND `content NOT LIKE '{name}:%'`. The content prefix check exists for backward compatibility with messages stored before the `is_bot_message` column migration.

**Risk:** Permanent query overhead and risk of false positives (user messages starting with bot name prefix). Backfill ran at migration time (`UPDATE messages SET is_bot_message = 1 WHERE content LIKE ?`), so the prefix check is theoretically unnecessary for all existing rows.

**Recommendation:** After confirming all existing messages are backfilled, remove the `content NOT LIKE` clause.

### 5.5.10 Shutdown Doesn't Wait for Active Work

**Current state:** `queue.shutdown()` sets a flag and detaches active containers but doesn't wait for them to complete. The IPC watcher and scheduler are not explicitly stopped. The database is not explicitly closed.

**Risk:** Race condition between process exit and container cleanup. IPC files written by containers after orchestrator exits are orphaned until next startup.

**Recommendation:** Wait for active containers to produce their final output (with a timeout), stop subsystem loops explicitly, and close the database connection.

---

## 5.6 Implicit Assumptions

These assumptions are not documented in code but are critical to correct operation:

| Assumption | Where It Matters | What Breaks If Wrong |
|------------|-----------------|---------------------|
| Node.js event loop is single-threaded | `group-queue.ts` concurrency checks (no mutex) | Race conditions in container slot allocation |
| Agent produces valid JSON between output markers | `container-runner.ts:316` (direct `JSON.parse`) | Silent output loss (caught, logged, continued) |
| Agent exits on stdin EOF | Idle timeout and task close delay | Container hangs until hard timeout (30 min) |
| Docker `--rm` flag always works | `group-queue.ts` shutdown (containers detached) | Orphaned containers accumulate |
| WhatsApp JID formats are stable | `whatsapp.ts` JID translation, `ownsJid()` | Messages mis-routed or dropped |
| SQLite writes are fast enough for event loop | All `db.ts` synchronous calls | Event loop blocked, polling delays |
| `~/.config/nanoclaw/` is writable by process user | Mount allowlist loading | All additional mount validation fails |
| Container filesystem is ephemeral | Agent runner writes to `/tmp/` | Stale state if containers reused (mitigated by `--rm`) |
| Cron expressions use the system timezone | `ipc.ts:217`, `task-scheduler.ts:188` | Tasks fire at wrong times |
| Group metadata `subject` field always exists | `whatsapp.ts:276` | Sync fails for groups without subjects |
