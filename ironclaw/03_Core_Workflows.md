# 03 — Core Workflows

## 3.1 Workflow 1: Chat → Job Execution → Response

The primary user journey. A message arrives from any channel, is routed through the agent loop, optionally spawns parallel jobs, executes tools inside sandboxes, and streams results back in real time.

---

### Phase 1: Message Ingestion

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Sends message via any channel   Channel wraps as IncomingMessage    —
                (TUI / Web / Webhook / WASM)    { id, channel, user_id,
                                                  thread_id, content, metadata }

System          BeforeInbound hook fires        Hook may modify/reject message      —
                                                If rejected: stop processing

System          Parse submission type           SubmissionParser::parse() detects:  —
                                                UserInput, SystemCommand, Undo,
                                                Redo, Compact, Clear, NewThread,
                                                Heartbeat, Quit, Interrupt,
                                                ExecApproval, ApprovalResponse,
                                                SwitchThread, Resume, Summarize,
                                                Suggest

System          Resolve session & thread        SessionManager::resolve_thread()    sessions HashMap
                                                builds ThreadKey(user, channel,     thread_map HashMap
                                                ext_thread_id) → internal UUID.     undo_managers HashMap
                                                Creates session/thread if new.
                                                Fires OnSessionStart hook.

System          Check pending auth mode         If thread awaiting auth token:      —
                                                route to process_auth_token()
                                                instead of normal flow
```

### Phase 2: Agentic Reasoning Loop

For `UserInput` submissions, the dispatcher runs the agentic loop (`run_agentic_loop`):

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Load workspace system prompt    Fetch identity files:               —
                                                IDENTITY.md, SOUL.md, AGENTS.md,
                                                USER.md (injected into system
                                                prompt). Skip MEMORY in group chat.

System          Select active skills            Gating: check bin/env/config        —
                                                requirements via spawn_blocking.
                                                Scoring: keywords(+10 exact,
                                                +5 substring, cap 30), tags(+3,
                                                cap 15), regex(+20, cap 40).
                                                Budget: fit top-scoring skills
                                                into SKILLS_MAX_TOKENS budget
                                                (token cost ≈ bytes × 0.25).

System          Build LLM context               System prompt + skill context       —
                                                (XML blocks with trust labels) +
                                                conversation history + user msg.

System          Call LLM for reasoning          LlmProvider::respond_with_tools()   llm_calls table
                                                Returns either Text or ToolCalls.   (provider, model,
                                                On ContextLengthExceeded: auto-     tokens, cost)
                                                compact (keep system + last user
                                                + current turn) and retry once.

System          Apply tool attenuation          If any installed skill active:      —
                                                drop tool list to read-only
                                                subset. Log removed tools.
```

### Phase 3: Tool Execution Pipeline

When the LLM returns tool calls:

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Preflight (sequential)          For each tool call:                 —
                                                1. Fire BeforeToolCall hook
                                                   (may modify params or reject)
                                                2. Check approval requirement:
                                                   Never → proceed
                                                   UnlessAutoApproved → check
                                                     session auto_approved_tools
                                                   Always → must get approval
                                                3. If approval needed: stop,
                                                   store PendingApproval, defer
                                                   remaining calls

System          Broadcast ToolStarted           SSE/WS event to all clients         —

System          Execute tools (parallel)        If 1 tool: execute inline           job_actions table
                                                If N tools: JoinSet::spawn()        (input, output_raw,
                                                per tool with indexed collection.    output_sanitized,
                                                                                    sanitization_warnings,
                Per-tool pipeline:                                                   cost, duration_ms,
                                                                                    success/error)
                1. Rate limit check
                   (per-tool, per-user)
                2. Validate params via
                   SafetyLayer::validator()
                3. Execute with timeout:
                   tokio::time::timeout(
                     tool_timeout,
                     tool.execute(params, ctx)
                   )
                4. Sanitize output via
                   SafetyLayer::sanitize()
                5. Wrap for LLM:
                   <tool_output name="X"
                    sanitized="true">

System          Broadcast ToolCompleted         SSE/WS event with success/fail      —

System          Post-flight (sequential,        For each result in original order:   thread.turns
                original order)                 - Record in thread turn history       (tool_calls vec)
                                                - Check for auth_required
                                                - Add tool_result to LLM context
                                                - Broadcast ToolResult status

System          Loop or respond                 If tools executed: loop back to      —
                                                LLM for next reasoning step.
                                                If text response: check for
                                                completion signals. If auth
                                                needed: enter auth mode. If
                                                approval pending: wait.
```

### Phase 4: Iteration Control & Completion

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Check iteration limits          Hard ceiling: 500 iterations         —
                                                Soft limit: max_tool_iterations
                                                (default 50, from config).
                                                At limit-1: inject nudge message.
                                                At limit: force text-only response.

System          Detect completion signals       Word-boundary patterns checked:      —
                                                ✓ "job is complete", "task is
                                                  done", "completed successfully"
                                                ✗ "not complete", "incomplete"
                                                ✗ Bare words in other contexts

System          Return response                 Text sent to channel manager         conversation_messages
                                                                                    table

System          BeforeOutbound hook fires       Hook may modify/suppress response   —

System          Broadcast to clients            SSE Response event + WebSocket      —
                                                frame to all connected clients.
                                                Silent replies suppressed.

System          Check event triggers            routine_engine                       routines table
                                                .check_event_triggers()             routine_runs table
                                                fires matching routines async.
```

### Phase 5: Approval Interrupt (when triggered)

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Store pending approval          PendingApproval {                   thread.pending_approval
                                                  request_id, tool_name,
                                                  parameters, description,
                                                  context_messages,
                                                  deferred_tool_calls
                                                }

System          Broadcast ApprovalNeeded        SSE/WS event to all clients         —

User            Responds with approval          Via TUI overlay, Web POST            —
                                                /api/chat/approval, or WS msg.
                                                Options: approve / always / deny

System          Process approval                If "always": add tool to             session.auto_approved
                                                  auto_approved_tools set.           _tools HashSet
                                                If approved: resume execution
                                                  of deferred tool calls.
                                                If denied: return ToolError to
                                                  LLM, agent adapts.
```

---

## 3.2 Workflow 2: Skill Discovery → Install → Activation

The skill lifecycle from registry search through trust-gated prompt injection.

---

### Phase 1: Discovery

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Invokes skill_search tool       ClawHub catalog client sends         —
                with query string               GET /api/v1/search?q={query}
                                                to registry (default:
                                                clawhub.ai). Cache TTL: 5 min.
                                                Max results: 25.

System          Return search results           List of SkillEntry with:             —
                                                name, version, description,
                                                author, download count,
                                                trust indicators.

User            Browses via skill_list tool     Lists all discovered skills          —
                                                from filesystem directories +
                                                installed registry skills.
                                                Shows: name, trust level,
                                                activation keywords, status.
```

### Phase 2: Installation

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Invokes skill_install with      Catalog client downloads SKILL.md    filesystem:
                skill name                      from registry.                       ~/.ironclaw/
                                                                                    installed_skills/
                                                                                    {name}/SKILL.md

System          Parse SKILL.md                  Parser extracts:                     —
                                                - YAML frontmatter (name, version,
                                                  description, activation config,
                                                  metadata with requirements)
                                                - Markdown body (prompt content)

System          Assign trust level              Source directory determines trust:    —
                                                ~/.ironclaw/skills/ → Trusted
                                                <workspace>/skills/ → Trusted
                                                ~/.ironclaw/installed_skills/ →
                                                  Installed (read-only tools)

System          Compile patterns                Pre-compile regex patterns,          SkillRegistry
                                                lowercase keywords and tags          in-memory cache
                                                for scoring performance.

System          Register skill                  Add to SkillRegistry with            —
                                                LoadedSkill { manifest, body,
                                                compiled_patterns, trust_level }
```

### Phase 3: Activation (Per-Turn)

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Sends a message                 Message enters agentic loop          —

System          Gating check                    For each registered skill:           —
                (spawn_blocking)                - bins: which/where lookup
                                                - env: std::env::var check
                                                - config: path existence check
                                                Skills with unmet prereqs skipped.

System          Deterministic scoring           Score message against each skill:    —
                                                - Keyword exact match: +10
                                                - Keyword substring: +5
                                                  (cap: 30 total)
                                                - Tag match: +3 each (cap: 15)
                                                - Regex match: +20 each (cap: 40)
                                                Max possible score: 85

System          Budget fitting                  Sort by score DESC.                  —
                                                For each, estimate token cost:
                                                  declared max_context_tokens
                                                  OR body.len() × 0.25
                                                  (min 1 token).
                                                Add if fits in remaining budget.
                                                Skip if exceeds.

System          Tool attenuation                Check trust of all active skills:    tool definitions
                                                - All trusted → full tool access     (filtered list)
                                                - Any installed → ceiling drops
                                                  to: memory_search, memory_read,
                                                  memory_tree, time, echo, json,
                                                  skill_list, skill_search

System          Inject into LLM context         Active skill bodies injected as      —
                                                XML blocks with trust labels
                                                into system prompt before LLM
                                                reasoning call.
```

### Phase 4: Removal

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Invokes skill_remove with       SkillRegistry removes from           filesystem:
                skill name                      registry and deletes files           delete SKILL.md
                                                from installed_skills directory.      SkillRegistry cache
```

---

## 3.3 Workflow 3: Routine Lifecycle

Automated task execution via cron schedules, event patterns, webhooks, or manual triggers.

---

### Phase 1: Routine Creation

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Invokes routine_create tool     Validate trigger config:             routines table
                with name, trigger, action,     - Cron: parse schedule string
                guardrails, notify config       - Event: compile regex pattern
                                                - Webhook: validate path/secret
                                                - Manual: no config needed
                                                Validate action config:
                                                - Lightweight: prompt + paths
                                                - FullJob: title + description
                                                Compute next_fire_at for cron.
                                                Save to DB.
```

### Phase 2: Trigger Evaluation

#### Cron Triggers

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Cron ticker fires               Spawned at startup with              —
                (every N seconds,               configurable interval.
                configurable)                   Skips immediate first tick.

System          Load due routines               DB query: enabled=true AND           —
                                                next_fire_at <= now

System          Gate checks (per routine)       1. Cooldown: last_run_at +           —
                                                   cooldown_secs > now? → skip
                                                2. Concurrent: count running
                                                   runs < max_concurrent?
                                                3. Global capacity: total
                                                   running < config max?

System          Spawn execution                 Create RoutineRun record.            routine_runs table
                                                Spawn async task.
```

#### Event Triggers

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Sends any message               After normal response processing,    —
                                                check_event_triggers() called.

System          Match against event routines    For each event routine:              —
                                                1. Channel filter: skip if
                                                   routine.channel != msg.channel
                                                   (None = match all)
                                                2. Pattern match: compiled regex
                                                   tested against message content
                                                3. Same gate checks as cron

System          Fire matching routines          Spawn async execution for each       routine_runs table
                                                match. Return count fired.
```

#### Webhook Triggers

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
External        HTTP POST to webhook endpoint   Validate HMAC secret if              —
                                                configured. Route to matching
                                                routine by path.

System          Fire routine                    Same gate + spawn flow               routine_runs table
```

#### Manual Triggers

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Invokes routine tool or         fire_manual(): check enabled,        routine_runs table
                POST /api/routines/{id}/trigger check concurrent limit.
                                                Create RoutineRun, save to DB.
                                                Execute inline (caller waits
                                                for completion).
```

### Phase 3: Routine Execution

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Increment running counter       Atomic counter incremented.          running_count
                                                Decremented on completion            (AtomicU32)
                                                (survives panics via guard).

System          Route by action type            Lightweight → execute_lightweight()  —
                                                FullJob → execute_full_job()
```

#### Lightweight Execution

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Load context from workspace     Read files at declared               —
                                                context_paths (e.g.
                                                "context/priorities.md")

System          Load routine state              Read workspace path:                 —
                                                routines/{safe_name}/state.md

System          Build prompt                    Base prompt + context content +       —
                                                routine state + "ROUTINE_OK"
                                                sentinel instruction

System          Call LLM                        Single LLM call with                 llm_calls table
                                                max_tokens from action config
                                                (default 4096).

System          Evaluate response               If contains "ROUTINE_OK":            —
                                                → status = Ok, no notification.
                                                If empty/truncated:
                                                → status = Failed.
                                                Otherwise:
                                                → status = Attention
                                                  (actionable findings).
```

#### FullJob Execution

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Dispatch job via scheduler      scheduler.dispatch_job() creates     agent_jobs table
                                                full AgentJob with title,
                                                description, metadata.

System          Link run to job                 store.link_routine_run_to_job()      routine_runs.job_id

System          Job executes independently      Follows Workflow 1 (Chat → Job)      (per Workflow 1)
                                                with full tool access.
```

### Phase 4: Completion & Notification

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Complete run record              store.complete_routine_run()         routine_runs
                                                with status, summary,                (status, completed_at,
                                                tokens_used.                         result_summary,
                                                                                    tokens_used)

System          Update routine state            Update last_run_at, next_fire_at     routines table
                                                (recalculated from cron schedule),   (last_run_at,
                                                increment run_count.                  next_fire_at,
                                                On failure: increment                 run_count,
                                                consecutive_failures.                 consecutive_failures)
                                                On success: reset to 0.

System          Send notification               Gate by notify config:               —
                (if configured)                 - on_success: notify if Ok
                                                - on_attention: notify if Attention
                                                - on_failure: notify if Failed
                                                Send via notify_tx channel
                                                to configured channel/user.
                                                Icon: ✅ ok, 🔔 attention,
                                                ❌ failure.
```

---

## 3.4 Supporting Workflows

### Self-Repair (Background)

Runs on a periodic timer (default: every 60 seconds).

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Detect stuck jobs               context_manager.find_stuck_jobs()    —
                                                checks for JobState::Stuck.
                                                Calculates stuck_duration from
                                                started_at.

System          Attempt repair                  If repair_attempts >= max (3):       —
                (per stuck job)                 → ManualRequired, skip.
                                                Otherwise: call
                                                ctx.attempt_recovery()
                                                (state Stuck → InProgress).

System          Detect broken tools             store.get_broken_tools(threshold=5)  —
                                                Returns tools with ≥5 failures.

System          Attempt tool repair             Construct BuildRequirement with      dynamic_tools table
                (per broken tool)               error context. Call builder.build()  tool_failures table
                                                to regenerate WASM module.
                                                On success: mark repaired.
                                                On failure: increment attempts.
```

### Context Compaction

Triggered automatically when context approaches token limits, or manually via `/compact` command.

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Monitor context pressure        ContextMonitor estimates tokens:      —
                                                words × 1.3 + 4 overhead per msg.
                                                Limit: 100,000 tokens (default).
                                                Threshold: 80% triggers action.

System          Select strategy                 Based on overage ratio:              —
                                                > 95%: Truncate (keep 3 turns)
                                                > 85%: Summarize (keep 5 turns)
                                                ≤ 85%: MoveToWorkspace (keep 10)

System          Execute compaction              Summarize: LLM generates summary     memory_documents
                                                (max_tokens=1024, temp=0.3).         (daily/{date}.md)
                                                Write to workspace daily log.
                                                Truncate thread turns.
                                                MoveToWorkspace: format old turns,
                                                append to daily log, truncate.
                                                Truncate: drop old turns directly.

System          Return result                   CompactionResult {                   thread.turns
                                                  turns_removed,                     (truncated)
                                                  tokens_before,
                                                  tokens_after,
                                                  summary_written,
                                                  summary
                                                }
```

### Session Lifecycle

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
System          Session creation                On first message from a user.        sessions HashMap
                                                Fires OnSessionStart hook.

System          Thread mapping                  ThreadKey(user, channel,             thread_map HashMap
                                                ext_thread_id) → stable UUID.
                                                Same key always maps to same
                                                thread. Stale mappings detected
                                                and refreshed.

System          Session pruning                 Every 10 minutes. Find sessions      sessions HashMap
                (background timer)              with last_active_at < cutoff         thread_map HashMap
                                                (default idle timeout: 7 days).      undo_managers HashMap
                                                Fire OnSessionEnd hooks.
                                                Remove from all maps.
                                                Warning logged at ≥1000 sessions.
```

### Undo/Redo

```
Actor           Action                          System Response                     Data Updated
─────           ──────                          ───────────────                     ────────────
User            Types "undo" or "/undo"         UndoManager::undo() restores         thread.turns
                                                previous turn checkpoint.            undo_managers
                                                Returns undone turn content.

User            Types "redo" or "/redo"         UndoManager::redo() re-applies       thread.turns
                                                the undone turn.                     undo_managers
                                                Returns redone turn content.
```

---

## 3.5 Workflow Decision Tree

Summary of how an incoming message is routed:

```
Message arrives
│
├─ Parse submission type
│
├─ Quit?               → Shutdown
├─ SystemCommand?      → Handle command (e.g., /job, /status, /list)
├─ Undo/Redo?          → UndoManager operation
├─ Compact?            → Context compaction
├─ Clear?              → Clear thread history
├─ NewThread?          → Create new thread, switch active
├─ SwitchThread?       → Switch to existing thread
├─ Resume?             → Resume paused thread
├─ Interrupt?          → Cancel in-progress turn
├─ Heartbeat?          → Trigger heartbeat check
├─ Summarize?          → Generate conversation summary
├─ Suggest?            → Generate follow-up suggestions
├─ ExecApproval?       → Process tool approval with request_id
├─ ApprovalResponse?   → Process simple yes/no/always approval
│
└─ UserInput?          → Enter agentic loop:
                          1. Load workspace identity
                          2. Select & gate skills
                          3. Build LLM context
                          4. Loop: LLM reason → execute tools → repeat
                          5. Return text response
                          6. Check event triggers
```
