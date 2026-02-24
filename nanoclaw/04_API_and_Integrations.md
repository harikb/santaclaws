# 04 — API & Agent Capabilities

NanoClaw has no HTTP API. All interaction flows through two interfaces: **WhatsApp messages** (user-facing) and **file-based IPC** (agent-facing). This document describes both, plus the full set of tools available to agents inside containers.

---

## 4.1 API Strategy

There are no REST endpoints, GraphQL schemas, or WebSocket servers. The system's "API" is:

| Interface | Direction | Protocol | Used By |
|-----------|-----------|----------|---------|
| WhatsApp messages | User → System | Baileys (WhatsApp Web) | Humans in WhatsApp groups |
| Container stdin | System → Agent | JSON over stdin pipe | Agent runner inside container |
| Container stdout | Agent → System | JSON wrapped in text markers | Agent runner inside container |
| IPC files (messages) | Agent → System → User | JSON files, atomic write | MCP server inside container |
| IPC files (tasks) | Agent → System | JSON files, atomic write | MCP server inside container |
| IPC files (input) | System → Agent | JSON files, polled by agent | Orchestrator piping follow-ups |

---

## 4.2 Channel Abstraction

All messaging channels implement a common interface. Currently only WhatsApp is built-in, but the abstraction supports adding Telegram, Discord, etc.

### Channel Interface

```typescript
interface Channel {
  name: string;
  connect(): Promise<void>;
  sendMessage(jid: string, text: string): Promise<void>;
  isConnected(): boolean;
  ownsJid(jid: string): boolean;
  disconnect(): Promise<void>;
  setTyping?(jid: string, isTyping: boolean): Promise<void>;
}
```

### Inbound Callbacks

```typescript
// Delivers parsed message to orchestrator
type OnInboundMessage = (chatJid: string, message: NewMessage) => void;

// Notifies orchestrator of chat existence (for discovery)
type OnChatMetadata = (
  chatJid: string,
  timestamp: string,
  name?: string,
  channel?: string,    // "whatsapp" | "telegram" | "discord"
  isGroup?: boolean,
) => void;
```

### WhatsApp Channel

| Method | Behavior |
|--------|----------|
| `connect()` | Establishes Baileys connection. Displays QR code on first auth. Handles reconnection with exponential backoff. |
| `sendMessage(jid, text)` | Sends text to chat. Prefixes with `{ASSISTANT_NAME}: ` if sharing a phone number. Queues messages if disconnected; flushes on reconnect. |
| `setTyping(jid, isTyping)` | Sends composing/paused presence update. |
| `syncGroupMetadata(force?)` | Fetches all group names from WhatsApp. 24-hour cache. Updates `chats` table. |
| `ownsJid(jid)` | Returns true for `*@g.us` and `*@s.whatsapp.net` patterns. |

---

## 4.3 Container Input/Output Protocol

### Input (stdin → container)

The orchestrator writes a single JSON object to the container's stdin, then closes it.

```typescript
interface ContainerInput {
  prompt: string;                     // Formatted XML messages or task prompt
  sessionId?: string;                 // Resume existing Claude session
  groupFolder: string;                // "main" | "family-chat" | etc.
  chatJid: string;                    // WhatsApp JID
  isMain: boolean;                    // Privilege flag
  isScheduledTask?: boolean;          // true for scheduler-invoked runs
  assistantName?: string;             // Trigger word / bot name
  secrets?: Record<string, string>;   // API keys (deleted after write)
}
```

**Secrets handled:** `CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_API_KEY`. Passed via stdin only — never written to disk, env vars, or logs. The `secrets` field is deleted from the input object immediately after writing to stdin.

### Output (container → stdout)

The agent emits results wrapped in text markers. Multiple marker pairs can be emitted (streaming).

```
---NANOCLAW_OUTPUT_START---
{"status":"success","result":"Here is the answer...","newSessionId":"sess_abc123"}
---NANOCLAW_OUTPUT_END---
```

```typescript
interface ContainerOutput {
  status: 'success' | 'error';
  result: string | null;              // Agent's response text
  newSessionId?: string;              // Updated session ID for persistence
  error?: string;                     // Error details (when status = 'error')
}
```

**Streaming behavior:** Each marker pair triggers the `onOutput` callback immediately. The orchestrator strips `<internal>...</internal>` tags, then sends the cleaned text to WhatsApp. Hard timeout resets on each output marker.

### Follow-Up Input (orchestrator → container, while running)

When a container is idle-waiting and new messages arrive, they're written as JSON files to the container's IPC input directory:

```
data/ipc/{group}/input/{timestamp}.json
```

```json
{ "type": "message", "text": "<messages>...</messages>" }
```

The agent runner polls this directory every 500ms. A close sentinel file (`_close`) signals the container to shut down gracefully.

---

## 4.4 Agent Tools (MCP Server)

Inside each container, a dedicated MCP server (`ipc-mcp-stdio.ts`) exposes NanoClaw-specific tools under the `mcp__nanoclaw__*` namespace. These are the "API" that agents use to interact with the outside world.

### send_message

Send a message to the user or group immediately, without waiting for the agent turn to complete.

```
send_message(text: string, sender?: string) → void
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `text` | string | Yes | Message body |
| `sender` | string | No | Identity label (e.g., "Researcher"). Used by Telegram for multi-bot display. |

**Authorization:** Main group can send to any chat. Non-main groups can only send to their own chat.

**Use case:** Progress updates, multi-part responses, proactive notifications during long-running tasks.

---

### schedule_task

Create a recurring or one-shot scheduled task.

```
schedule_task(
  prompt: string,
  schedule_type: 'cron' | 'interval' | 'once',
  schedule_value: string,
  context_mode?: 'group' | 'isolated',
  target_group_jid?: string
) → { task_id: string }
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | Yes | Instructions for the agent when the task fires |
| `schedule_type` | enum | Yes | `cron`, `interval`, or `once` |
| `schedule_value` | string | Yes | Cron expression, milliseconds, or ISO 8601 local timestamp (no `Z` suffix) |
| `context_mode` | enum | No | `'group'` (reuse session, default) or `'isolated'` (fresh each run) |
| `target_group_jid` | string | No | Main group only — target a different group's chat |

**Schedule value examples:**

| Type | Value | Meaning |
|------|-------|---------|
| `cron` | `"0 9 * * *"` | Daily at 9:00 AM |
| `cron` | `"*/5 * * * *"` | Every 5 minutes |
| `interval` | `"3600000"` | Every hour |
| `interval` | `"300000"` | Every 5 minutes |
| `once` | `"2026-02-23T15:30:00"` | One-time at local time |

---

### list_tasks

List all scheduled tasks visible to the current group.

```
list_tasks() → string (formatted task list)
```

**Authorization:** Main group sees all tasks. Non-main groups see only their own.

**Output format:** Human-readable list with task ID, prompt excerpt, schedule, status, and next run time.

---

### pause_task

Pause an active task. It won't run until resumed.

```
pause_task(task_id: string) → void
```

**Authorization:** Own tasks, or any task if main group.

---

### resume_task

Resume a paused task.

```
resume_task(task_id: string) → void
```

**Authorization:** Own tasks, or any task if main group.

---

### register_group

Register a new WhatsApp group for agent processing.

```
register_group(
  jid: string,
  name: string,
  folder: string,
  trigger: string
) → void
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `jid` | string | WhatsApp group JID (from `available_groups.json`) |
| `name` | string | Display name |
| `folder` | string | Filesystem folder name (alphanumeric, hyphens, max 64 chars) |
| `trigger` | string | Trigger pattern (e.g., `"@Andy"`) |

**Authorization:** Main group only. Non-main groups receive an error.

**Context:** The agent reads `/workspace/ipc/available_groups.json` (written by orchestrator before each run) to discover unregistered groups and their JIDs.

---

### cancel_task

Permanently delete a scheduled task and its run history.

```
cancel_task(task_id: string) → void
```

**Authorization:** Own tasks, or any task if main group.

---

## 4.5 Built-In Agent Tools (Claude SDK)

Beyond the MCP tools, agents have access to the full Claude Code SDK toolset:

| Tool | Purpose |
|------|---------|
| `Bash` | Execute shell commands (with secrets sanitization hook) |
| `Read` | Read files |
| `Write` | Create/overwrite files |
| `Edit` | Targeted string replacement in files |
| `Glob` | Find files by pattern |
| `Grep` | Search file contents |
| `WebSearch` | Search the web |
| `WebFetch` | Fetch and analyze web pages |
| `Task` / `TaskOutput` / `TaskStop` | Spawn sub-agents |
| `TeamCreate` / `TeamDelete` / `SendMessage` | Agent swarm coordination |
| `TodoWrite` | Internal task tracking |
| `ToolSearch` | Discover available MCP tools |
| `Skill` | Invoke registered skills |
| `NotebookEdit` | Edit Jupyter notebooks |

**Permission mode:** `bypassPermissions` — agents run without interactive approval prompts.

**Bash sanitization hook:** A `PreToolUse` hook prepends `unset ANTHROPIC_API_KEY CLAUDE_CODE_OAUTH_TOKEN` to all Bash commands, preventing secrets from leaking to shell subprocesses.

---

## 4.6 Browser Automation (agent-browser)

Agents have full browser automation via the `agent-browser` CLI tool, backed by system Chromium.

### Navigation

```bash
agent-browser open <url>           # Navigate to URL
agent-browser back                 # History back
agent-browser forward              # History forward
agent-browser reload               # Reload page
agent-browser close                # Close browser
```

### Page Analysis

```bash
agent-browser snapshot             # Full accessibility tree
agent-browser snapshot -i          # Interactive elements only (recommended)
agent-browser snapshot -c          # Compact output
agent-browser snapshot -d 3        # Limit depth
agent-browser snapshot -s "#main"  # Scope to CSS selector
```

### Interactions (use `@ref` identifiers from snapshot output)

```bash
agent-browser click @e1            # Click element
agent-browser fill @e2 "text"      # Clear + type
agent-browser type @e2 "text"      # Type without clearing
agent-browser press Enter          # Keyboard shortcut
agent-browser select @e1 "value"   # Dropdown selection
agent-browser scroll down 500      # Scroll page
agent-browser upload @e1 file.pdf  # File upload
```

### Data Extraction

```bash
agent-browser get text @e1         # Element text content
agent-browser get html @e1         # Element innerHTML
agent-browser get value @e1        # Input value
agent-browser get attr @e1 href    # Element attribute
agent-browser get title            # Page title
agent-browser get url              # Current URL
agent-browser get count ".item"    # Count matching elements
```

### Screenshots & Export

```bash
agent-browser screenshot           # Save to temp directory
agent-browser screenshot --full    # Full page capture
agent-browser pdf output.pdf       # Export as PDF
```

### Session Persistence

```bash
agent-browser state save auth.json   # Save login cookies/state
agent-browser state load auth.json   # Restore session
```

---

## 4.7 Integration Contracts

### WhatsApp (via Baileys)

| Aspect | Detail |
|--------|--------|
| **Library** | `@whiskeysockets/baileys` v7.0.0-rc.9 |
| **Auth** | Multi-file auth state in `store/auth/`. QR code displayed on first connect. |
| **Inbound** | `messages.upsert` socket event. Extracts: JID, timestamp, sender, content, bot flag. |
| **Outbound** | `sock.sendMessage(jid, { text })`. Auto-prefixed with bot name on shared numbers. |
| **Presence** | `sock.sendPresenceUpdate('composing'/'paused', jid)` for typing indicators. |
| **Group sync** | `sock.groupFetchAllParticipating()` → updates `chats` table. 24h cache. |
| **JID format** | Groups: `*@g.us`. Individuals: `*@s.whatsapp.net`. LID translation for multi-device. |
| **Reconnection** | Automatic with retry. Queued messages flushed on reconnect. |

### Claude Agent SDK

| Aspect | Detail |
|--------|--------|
| **Runtime** | Runs inside Docker/Apple Container as `node` user |
| **Session** | Persistent per group via `sessionId`. Stored in `data/sessions/{group}/.claude/`. |
| **Agent swarms** | Enabled via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings. |
| **Auto-memory** | Enabled via `CLAUDE_CODE_DISABLE_AUTO_MEMORY=0`. Per-group `CLAUDE.md` files. |
| **Permission mode** | `bypassPermissions` — no interactive prompts. |
| **Hooks** | `PreCompact` (transcript archiving), `PreToolUse` (Bash secrets sanitization). |
| **Secrets injection** | Via `options.env` in SDK query config (not `process.env`). |

### Docker / Apple Container

| Aspect | Detail |
|--------|--------|
| **Image** | `nanoclaw-agent:latest` (built from `container/Dockerfile`) |
| **Base** | `node:22-slim` + Chromium + git + curl |
| **Runtime binary** | `docker` (configurable via `CONTAINER_RUNTIME`) |
| **Container flags** | `-i` (interactive stdin), `--rm` (auto-cleanup), `--name nanoclaw-{folder}-{ts}` |
| **Entrypoint** | Compiles TypeScript → reads stdin → executes agent runner |
| **Orphan cleanup** | On startup: `docker ps --filter name=nanoclaw-` → stop any lingering containers |
| **User** | `node` (non-root, UID 1000) |

---

## 4.8 Context Files Written Before Each Run

The orchestrator writes snapshot files into the container's IPC directory before spawning:

### available_groups.json

Written to `/workspace/ipc/available_groups.json`. Provides the agent with group discovery data.

- **Main group:** sees all registered groups + unregistered WhatsApp groups (for onboarding).
- **Non-main groups:** sees only their own group entry.

### current_tasks.json

Written to `/workspace/ipc/current_tasks.json`. Provides the agent with scheduled task state.

- **Main group:** sees all tasks across all groups.
- **Non-main groups:** sees only tasks belonging to their group folder.

### sessions-index.json

Read (not written) by the agent from `/workspace/group/sessions-index.json`. Contains summaries of past conversation sessions, used for transcript archiving filenames.
