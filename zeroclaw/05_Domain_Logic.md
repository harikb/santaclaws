# 05 — Domain Logic & Debt

> ZeroClaw Technical Blueprint | File 5 of 5

---

## 1. Security Rules

### 1.1 Command Execution Policy

The agent's ability to execute shell commands is governed by a multi-layer policy engine. Every command passes through these gates in order:

```
Command string
    │
    ├─► 1. Autonomy gate: ReadOnly → reject unconditionally
    │
    ├─► 2. Subshell/redirect block: backtick, $(), ${}, <(), >(),
    │      unquoted <, >, bare &, tee → reject
    │
    ├─► 3. Quote-aware segment split: ;, |, &&, ||, \n
    │      (each segment validated independently)
    │
    ├─► 4. Per-segment allowlist check:
    │      - Skip env assignments (KEY=value prefix)
    │      - Extract base command (first token)
    │      - Must be in allowed_commands list (13 defaults)
    │
    ├─► 5. Dangerous argument scan:
    │      - find -exec / find -ok → reject
    │      - git config / git alias / git -c → reject
    │      - chmod 777 / chmod -R → reject
    │      - curl|sh / curl|bash → reject
    │
    ├─► 6. Risk classification:
    │      Low:    ls, cat, grep, find, echo, pwd, wc, head, tail, date
    │      Medium: git (commit/push/reset/merge/rebase/...), npm/cargo (install/publish/...), touch, mkdir, mv, cp, ln
    │      High:   rm, mkfs, dd, sudo, curl, wget, ssh, nc, chmod, chown + 16 more
    │
    ├─► 7. Risk enforcement:
    │      High + block_high_risk_commands=true → reject
    │      High + Supervised + !approved → reject
    │      Medium + Supervised + require_approval_for_medium_risk + !approved → reject
    │
    ├─► 8. Forbidden path argument scan:
    │      - Absolute paths in arguments → reject
    │      - --opt=/absolute/path → reject
    │      - ~otheruser expansion → reject
    │      - Input redirection from absolute path → reject
    │
    ├─► 9. Rate limit: actions_this_hour >= max_actions_per_hour → reject
    │
    └─► 10. Execute: sh -c "{command}" with env_clear() + 9 safe vars only
```

**Shell environment isolation:** Every command runs with `env_clear()`. Only 9 variables are passed through: `PATH`, `HOME`, `TERM`, `LANG`, `LC_ALL`, `LC_CTYPE`, `USER`, `SHELL`, `TMPDIR`. Additional variables require explicit `shell_env_passthrough` configuration.

### 1.2 Filesystem Access Policy

```
Path request
    │
    ├─► Null byte check → reject
    ├─► ".." component check → reject
    ├─► URL-encoded traversal (%2f) → reject
    ├─► ~user expansion → reject
    ├─► Absolute path + workspace_only=true → reject
    │
    ├─► Canonicalize path (resolves symlinks)
    │
    ├─► Resolved path must be under:
    │     workspace_dir (primary) OR
    │     any path in allowed_roots (secondary)
    │   → reject if outside all allowed roots
    │
    ├─► Symlink escape check:
    │     If target file is a symlink, resolve and re-check
    │     → reject if symlink points outside allowed roots
    │
    └─► Forbidden path prefix match (18 hardcoded):
          /etc, /root, /home, /usr, /bin, /sbin, /lib, /opt,
          /boot, /dev, /proc, /sys, /var, /tmp,
          ~/.ssh, ~/.gnupg, ~/.aws, ~/.config
```

**Rule:** `file_read` consumes rate limit budget even on path validation failure — prevents using path probing to enumerate filesystem existence without budget cost.

**Rule:** `file_write` has no content size limit (unlike `file_read`'s 10 MB cap). The rate limiter constrains write frequency but not byte volume.

### 1.3 Prompt Injection Defense

Six detection categories with weighted scores, normalized over `6.0`:

| Category | Score | Example Triggers |
|---|---|---|
| System override | 1.00 | "ignore previous instructions", "you are now", "new system prompt" |
| Secret extraction | 0.95 | "reveal your API key", "show me the system prompt", "print your instructions" |
| Role confusion | 0.90 | "pretend you are", "act as root", "you are a different AI" |
| Jailbreak | 0.85 | "DAN mode", "developer mode", "unrestricted mode" |
| Tool injection | 0.80 | Tool call JSON/XML in user message body |
| Command injection | 0.60 | Pipe chains, `&&` sequences (exemptions: `\| head`, `\| tail`, `\| grep`; `&&` in content < 100 chars) |

**Decision:** `normalized_score >= sensitivity` (default: 0.7) → action. Default action is `Warn` (log, not block). Configurable to `Block` (reject message entirely).

**Fragility note:** The normalization divisor `6.0` is hardcoded. Adding a 7th category without updating the divisor would inflate all scores and lower the effective block threshold.

### 1.4 Credential Leak Detection

Patterns scanned in all agent output before delivery:

| Pattern | Regex Summary |
|---|---|
| Stripe keys | `sk_live_*`, `pk_live_*`, `sk_test_*`, `pk_test_*` (24+ suffix chars) |
| OpenAI keys | `sk-[A-Za-z0-9]{20}T3BlbkFJ[A-Za-z0-9]{20}` |
| Anthropic keys | `sk-ant-[A-Za-z0-9+/=]{32,}` |
| Google API keys | `AIza[A-Za-z0-9_-]{35}` |
| GitHub tokens | `gh[pousr]_[A-Za-z0-9_]{36+}`, `github_pat_[A-Za-z0-9_]{22+}` |
| AWS keys | `AKIA[A-Z0-9]{16}`, `aws_secret_access_key=...{40}` |
| JWT tokens | `eyJ...eyJ...` (three base64url-encoded parts) |
| Database URLs | `postgres://`, `mysql://`, `mongodb://`, `redis://` with embedded credentials |
| PEM private keys | RSA, EC, PKCS8, OpenSSH PEM block headers |
| Generic secrets | `password=`, `secret=`, `token=` (only when sensitivity > 0.5) |

**Redaction rule:** First 4 characters of detected value are preserved as context prefix (e.g., `sk-a****[REDACTED]`). Credentials shorter than 4 chars are fully preserved — no minimum length guard.

### 1.5 Domain Categories (E-Stop / OTP Gating)

Built-in domain categories for security gating:

| Category | Domains |
|---|---|
| `banking` | chase, bankofamerica, wellsfargo, fidelity, schwab, venmo, paypal, robinhood, coinbase |
| `medical` | mychart, epic, patient.portal.*, healthrecords.* |
| `government` | ssa.gov, irs.gov, login.gov, id.me |
| `identity_providers` | accounts.google.com, login.microsoftonline.com, appleid.apple.com |

**Rule:** `*.chase.com` matches `www.chase.com` but NOT bare `chase.com` — the wildcard requires at least one subdomain label.

**Rule:** `*` in patterns matches across DNS label boundaries — `*.example.com` matches `a.b.example.com` (multi-label), which may be broader than intended for some security use cases.

---

## 2. Cost & Budget Rules

### 2.1 Budget Enforcement

```
On every LLM call:
    │
    ├─► Calculate cost: (input_tokens / 1,000,000) * input_price
    │                  + (output_tokens / 1,000,000) * output_price
    │   Negative/invalid prices clamped to 0.0
    │
    ├─► Record to JSONL: workspace/state/costs.jsonl
    │
    ├─► Check budget (daily first, then monthly):
    │
    │   daily_cost >= daily_limit_usd ($10 default)
    │     → BudgetCheck::Exceeded (blocks further LLM calls)
    │
    │   monthly_cost >= monthly_limit_usd ($100 default)
    │     → BudgetCheck::Exceeded
    │
    │   daily_cost >= daily_limit * warn_at_percent/100 (80% default)
    │     → BudgetCheck::Warning
    │
    │   monthly_cost >= monthly_limit * warn_at_percent/100
    │     → BudgetCheck::Warning
    │
    └─► Otherwise: BudgetCheck::Allowed
```

**Rule:** `allow_override` flag (default: false) can bypass budget enforcement — this is a config-level escape hatch with no audit trail beyond the config change itself.

### 2.2 Hardcoded Model Pricing

These are compile-time constants — they do not auto-update when providers change pricing:

| Model | Input $/1M | Output $/1M |
|---|---|---|
| `anthropic/claude-sonnet-4-*` | 3.00 | 15.00 |
| `anthropic/claude-3.7-sonnet` | 3.00 | 15.00 |
| `anthropic/claude-3.5-sonnet` | 3.00 | 15.00 |
| `anthropic/claude-opus-4-*` | 15.00 | 75.00 |
| `anthropic/claude-3-opus` | 15.00 | 75.00 |
| `anthropic/claude-3-haiku` | 0.25 | 1.25 |
| `openai/gpt-4o` | 5.00 | 15.00 |
| `openai/gpt-4o-mini` | 0.15 | 0.60 |
| `openai/o1-preview` | 15.00 | 60.00 |
| `google/gemini-2.0-flash` | 0.10 | 0.40 |
| `google/gemini-1.5-pro` | 1.25 | 5.00 |

Models not in this table are unpriced — their usage is tracked but cost is recorded as $0.00. User-configurable `cost.prices` map can override or extend.

### 2.3 Rate Limiting

| Surface | Limit | Window | Budget Type |
|---|---|---|---|
| Tool actions | 20/hour (default) | 1-hour sliding window | Per SecurityPolicy instance |
| Pair endpoint | 10/min per IP | 60s sliding window | Per gateway |
| Webhook endpoint | 60/min per IP | 60s sliding window | Per gateway |
| Cost daily | $10/day (default) | Calendar day (UTC) | Per workspace |
| Cost monthly | $100/month (default) | Calendar month (UTC) | Per workspace |
| Cost daily (policy) | $5/day (500 cents) | Calendar day | Per SecurityPolicy |

---

## 3. Data Handling Rules

### 3.1 Memory

| Rule | Constraint |
|---|---|
| Memory key uniqueness | `UNIQUE` constraint on `key` column — upsert semantics |
| Hybrid search weights | 70% vector similarity + 30% keyword (BM25) by default |
| Minimum relevance score | 0.4 — entries below this are filtered from context injection |
| Memory context injection | Max 4 entries, max 800 chars per entry, max 4,000 chars total |
| Auto-save threshold | Messages ≥ 20 characters are auto-saved to memory |
| Auto-saved assistant responses | Filtered OUT of memory context injection (prevents echo loops) |
| Hygiene: archive | Entries older than 7 days moved to `archived` category |
| Hygiene: purge | Entries older than 30 days deleted |
| Conversation retention | Conversation-category entries older than 30 days deleted |
| Cold-boot hydration | If `brain.db` missing but `MEMORY_SNAPSHOT.md` exists: restore core memories automatically |
| Embedding cache | SHA-256 truncated to 8 bytes (16 hex chars) as cache key — collision probability ~1 in 2^64 |

### 3.2 Conversation History

| Rule | Constraint |
|---|---|
| Per-sender history | 50 messages max (FIFO eviction) |
| History compaction trigger | Non-system messages > `max_history_messages` (50) |
| Compaction: LLM-assisted | Oldest messages (up to 12,000 chars) summarized into ≤ 12 bullet points (max 2,000 chars) at temperature 0.2 |
| Compaction: fallback | Keep last 20 messages if LLM summarization fails |
| Channel compaction | Keep last 12 turns, truncate each to 600 chars |
| Context overflow | Consecutive same-role messages merged before sending to provider |
| Draft streaming threshold | Chunks accumulated to ≥ 80 chars before sending draft update |
| Outbound message cap | 20,000 chars max before delivery |

### 3.3 Cron Jobs

| Rule | Constraint |
|---|---|
| Shell job timeout | 120 seconds |
| Run history retention | 50 runs per job (pruned on new insert) |
| Run output truncation | 16,384 bytes max stored per run |
| One-shot auto-delete | `delete_after_run=true` + `Schedule::At` → deleted on success, disabled on failure |
| Agent job frequency warning | Jobs more frequent than 5 minutes emit warning log |
| Security policy failures | Not retried (deterministic — would fail again) |
| Retry backoff | Start: max(500ms, 200ms), double each attempt, cap at 30,000ms, jitter: `timestamp_subsec_millis % 250` |

### 3.4 Audit Log

| Rule | Constraint |
|---|---|
| Format | NDJSON (newline-delimited JSON) |
| Rotation | When file exceeds `max_size_mb` (default: 100 MB) |
| Backup count | Up to 10 numbered backups (`.1.log` through `.10.log`) |
| Event signing | Optional (`sign_events`, default: false) |

---

## 4. Provider Resilience Rules

### 4.1 Retry Logic

```
Primary provider call
    │
    ├─► Success → return
    │
    ├─► 4xx (except 429, 408) → non-retryable, fail immediately
    │
    ├─► 429 with business/quota error text → non-retryable, fail immediately
    │     Matched strings: "plan does not include", "insufficient balance",
    │     "insufficient quota", "quota exhausted", "out of credits",
    │     "no available package", "package not active", "purchase package",
    │     "model not available for your plan"
    │     Z.AI error codes 1113, 1311 → also non-retryable
    │
    ├─► 429 with Retry-After header → wait min(header_value, 30000ms), retry
    │
    ├─► Other retryable error → wait base_backoff_ms (min 50ms), retry
    │     Backoff doubles each attempt
    │
    └─► Exhausted retries → try fallback providers in order
          Each fallback resolves its own credentials independently
```

### 4.2 Three-Level Failover

```
Level 1: Model fallbacks (model_fallbacks config map)
    → Try alternative models with same provider

Level 2: Provider fallbacks (fallback_providers config list)
    → Try entirely different providers in order

Level 3: API key rotation (api_keys config list)
    → Round-robin across multiple API keys for same provider
    → AtomicUsize counter for lock-free rotation
```

### 4.3 Tool Call Parsing (7+ Format Fallbacks)

The agent loop parses LLM tool call responses through multiple format detectors, tried in sequence:

| Priority | Format | Description |
|---|---|---|
| 1 | Structured (native) | `response.tool_calls` array (OpenAI wire format) |
| 2 | JSON root object | `{"name": "...", "arguments": {...}}` or `{"name": "...", "parameters": {...}}` |
| 3 | JSON array | `[{"name": "...", "arguments": {...}}, ...]` |
| 4 | OpenAI nested | `{"function": {"name": "...", "arguments": "..."}, "id": "..."}` |
| 5 | `tool_calls` wrapper | `{"tool_calls": [...]}` |
| 6 | XML tag extraction | `<tool_call>...</tool_call>`, `<invoke name="...">...</invoke>` |
| 7 | MiniMax XML | `<invoke name="..."><parameter name="...">value</parameter></invoke>` |

**Tool name normalization aliases:** `bash`→`shell`, `fileread`/`read_file`→`file_read`, `filewrite`/`write_file`→`file_write`, `fileedit`/`edit_file`→`file_edit`, `search`→`content_search`, `glob`→`glob_search`

**XML meta-tags skipped** (not parsed as tool calls): `tool_call`, `toolcall`, `tool-call`, `invoke`, `thinking`, `thought`, `analysis`, `reasoning`, `reflection`

**Arguments accepted as:** JSON string (parsed) or JSON object (used directly). Both `"arguments"` and `"parameters"` keys are tried.

---

## 5. Validation Constraints

### 5.1 Config Validation

| Field | Constraint |
|---|---|
| `multimodal.max_images` | Clamped to `[1, 16]` |
| `multimodal.max_image_size_mb` | Clamped to `[1, 20]` |
| `web_search.max_results` | Clamped to `[1, 10]` |
| `sqlite_open_timeout_secs` | Capped at 300 seconds |
| `max_tool_iterations` | 0 falls back to 10 (not an error) |
| `gateway.port` | u16 range (1–65535) |
| `default_temperature` | f64 (no explicit range clamp) |
| `heartbeat.interval_minutes` | Minimum enforced as 5 minutes at runtime |
| `scheduler.poll_secs` | Minimum enforced as 5 seconds at runtime |

### 5.2 Input Validation

| Tool | Validation |
|---|---|
| `file_edit` | `old_string` must not be empty; must match exactly once (0 = "not found", 2+ = "ambiguous") |
| `shell` | Command string cannot be empty |
| `delegate` | `agent` name must exist in configured agents map; `prompt` must be non-empty |
| `http_request` | URL must start with `http://` or `https://`; no whitespace; domain must be in allowlist; no local/private hosts |
| `browser` | `action` must be one of 22 defined actions; `open` requires URL in domain allowlist |
| `cron_add` | Shell jobs require `command`; agent jobs require `prompt`; schedule must parse |
| `git_operations` | Arguments scanned for injection: `--exec=`, `--upload-pack=`, `--no-verify`, `-c`, shell metacharacters |
| `screenshot` | Filename sanitized: only basename used, shell-unsafe chars rejected (`'`, `"`, `` ` ``, `\`, `;`, `\|`, `&`, `()`, null, newline) |

### 5.3 Gateway Input Validation

| Endpoint | Validation |
|---|---|
| `POST /webhook` | Body ≤ 64 KB; must parse as JSON `{"message": string}` |
| `PUT /api/config` | Body ≤ 1 MB; must parse as valid TOML; `Config::validate()` called |
| `POST /pair` | `X-Pairing-Code` header required; rate-limited per IP |
| `POST /whatsapp` | `X-Hub-Signature-256` HMAC constant-time comparison |
| All `/api/*` | `Authorization: Bearer <token>` constant-time comparison |

---

## 6. Edge Cases & Implicit Behavior

### 6.1 Period Boundary Stale Cache (Cost Tracking)

Budget aggregates are cached and invalidated by comparing `Utc::now()` at check time. If two budget checks straddle midnight UTC, the first check uses stale aggregates. The staleness window is up to one polling interval (typically seconds). This means a brief window at midnight where the daily aggregate could be under-reported.

### 6.2 Approval Bypass on Non-CLI Channels

Interactive approval (`Yes/No/Always` prompt) only works on the CLI channel. On all non-CLI channels (Telegram, Discord, Slack, etc.), approval is **automatically granted** for all tools — the `ApprovalManager` returns `Yes` unconditionally. This is a design decision (async channels cannot block for user input), but means the `Supervised` autonomy level provides weaker guarantees on non-CLI channels.

### 6.3 Concurrent Sender Interruption

On interruptible channels (e.g., Telegram), a new message from the same sender+chat scope cancels the in-flight agent task via `CancellationToken`. The cancelled task produces no response. The cancellation is per-scope-key (`{channel}_{sender}_{reply_target}`), so messages in different threads or from different senders are independent.

### 6.4 FTS5 Trigger Sync

The `memories_fts` virtual table is synchronized via three SQLite triggers (`AFTER INSERT`, `AFTER DELETE`, `AFTER UPDATE`). If a write to the `memories` table succeeds but the trigger fails (e.g., FTS corruption), the main table and FTS index will be out of sync. There is no repair mechanism — manual `INSERT INTO memories_fts(memories_fts) VALUES('rebuild')` would be required.

### 6.5 Embedding Cache Hash Truncation

Embedding cache keys use SHA-256 truncated to 8 bytes (16 hex chars). While collision probability is astronomically low (~1 in 2^64), a collision would cause an incorrect cached embedding to be returned for a different input text. There is no integrity check on cache hits.

### 6.6 WebSocket Single-Turn Limitation

The `/ws/chat` endpoint is documented with `chunk`, `tool_call`, and `tool_result` frame types, but these are **not implemented**. Current behavior is single-turn request/response with no streaming and no tool call forwarding. Clients expecting streaming will receive only a `done` frame after the full response is computed.

### 6.7 Config Hot-Reload Race

Channel message processing checks config file modification time on every message. If the config file is being written concurrently (e.g., via `PUT /api/config`), the channel could read a partially-written file. The file write from the API is not atomic (no temp-file-then-rename pattern is evident).

### 6.8 Compact Context Mode

When `agent.compact_context = true` (intended for small models ≤ 13B parameters):
- Bootstrap file injection truncated to 6,000 chars (vs. normal 20,000)
- RAG chunk limit reduced to 2 (vs. normal default)
- Skills rendered in compact XML (no instructions or tool details)

---

## 7. Technical Debt Inventory

### 7.1 Zero Source-Level Debt Markers

A global search for `TODO`, `FIXME`, `HACK`, `XXX`, `WORKAROUND`, `DEPRECATED`, `TEMPORARY` across all Rust source files returned **zero matches**. The codebase carries no outstanding debt markers.

### 7.2 Architectural Debt

| Item | Location | Risk | Description |
|---|---|---|---|
| **Hardcoded model pricing** | `src/config/schema.rs` | Medium | Pricing table is a compile-time constant. Any provider price change requires code change + rebuild. No external update mechanism or TTL. User override exists via `cost.prices` config map. |
| **Hardcoded fallback model** | `src/gateway/mod.rs` | Low | `"anthropic/claude-sonnet-4"` fallback in gateway. When deprecated by provider, fails at runtime not config validation. |
| **Non-retryable 429 detection by substring** | `src/providers/reliable.rs` | Medium | Business/quota errors detected by matching error message text ("plan does not include", etc.). Provider API wording changes will cause these errors to fall through to retry loops. |
| **Z.AI-specific error codes** | `src/providers/reliable.rs` | Low | Error codes `1113` and `1311` hardcoded as non-retryable. Provider-specific, will silently break if Z.AI changes error schema. |
| **SQLite default timeout: infinite** | `src/memory/sqlite.rs` | Medium | Default `sqlite_open_timeout_secs = None` means WAL lock contention blocks forever. The 300s cap constant only applies to user-configured values exceeding it. |
| **No schema version tracking** | `src/memory/sqlite.rs` | Low | Migrations applied by column presence detection (`ALTER TABLE ADD COLUMN`). No version table. Future migrations require new detection logic per migration. |
| **Cost migration non-atomic** | `src/cost/tracker.rs` | Low | SQLite→JSONL migration uses `rename` with copy+delete fallback. Process kill between copy and delete orphans the old file. No checksum or completeness validation. |
| **Rate limiter O(N) eviction** | `src/gateway/mod.rs` | Low | LRU eviction scans all 10,000 keys linearly on overflow. Under sustained flood this is O(N) per request. |
| **file_write no size limit** | `src/tools/file_write.rs` | Low | No content size cap (unlike file_read's 10 MB). Rate limiter constrains frequency but not byte volume. A single large write could exhaust disk. |
| **Prompt guard normalization divisor** | `src/security/prompt_guard.rs` | Low | Hardcoded `6.0` divisor (number of categories). Adding a category without updating the divisor inflates all scores. |
| **OTel blocking HTTP client** | `Cargo.toml` | Low | OpenTelemetry uses `reqwest-blocking-client` to avoid Tokio reactor panics. This is a known impedance mismatch; future OTel upgrades may reintroduce the issue. |
| **MSRV constraint** | `Cargo.toml` | Low | MSRV 1.87 may prevent adoption of newer Rust features that would simplify code. |

### 7.3 Feature Flag Testing Risk

The following feature flags gate significant functionality with potentially untested interaction combinations:

| Flag | Scope |
|---|---|
| `sandbox-landlock` | Kernel-level filesystem sandbox |
| `sandbox-bubblewrap` | Userspace sandbox |
| `channel-matrix` | Matrix E2EE channel |
| `channel-lark` | Lark/Feishu WebSocket channel |
| `whatsapp-web` | WhatsApp Web protocol |
| `rag-pdf` | PDF text extraction |
| `memory-postgres` | PostgreSQL memory backend |
| `observability-otel` | OpenTelemetry export |
| `peripheral-rpi` | Raspberry Pi GPIO |
| `probe` | STM32 debug probe |
| `hardware` | USB + serial peripherals |
| `browser-native` | Rust-native browser automation |

No `all-features` CI validation path is visible in the build configuration, creating risk that certain feature combinations are not routinely tested.

### 7.4 Hardcoded Constants Summary

All hardcoded constants that encode business decisions, collected across the codebase:

| Constant | Value | Location |
|---|---|---|
| Gateway port | 42617 | `config/schema.rs` |
| Body size limit | 64 KB | `gateway/mod.rs` |
| Config body limit | 1 MB | `gateway/mod.rs` |
| Request timeout | 30s | `gateway/mod.rs` |
| Rate limiter sweep | 5 min | `gateway/mod.rs` |
| Rate limiter max keys | 10,000 | `gateway/mod.rs` |
| Idempotency TTL | 300s | `gateway/mod.rs` |
| Idempotency max keys | 10,000 | `gateway/mod.rs` |
| SSE buffer | 256 events | `gateway/mod.rs` |
| Shell timeout | 60s | `tools/shell.rs` |
| Shell output cap | 1 MB | `tools/shell.rs` |
| File read cap | 10 MB | `tools/file_read.rs` |
| PDF read cap | 50 MB | `tools/pdf_read.rs` |
| Image size cap | 5 MB | `tools/image_info.rs` |
| Screenshot base64 cap | 2 MB | `tools/screenshot.rs` |
| Content search timeout | 30s | `tools/content_search.rs` |
| Content search max results | 1,000 | `tools/content_search.rs` |
| Glob search max results | 1,000 | `tools/glob_search.rs` |
| Cron shell timeout | 120s | `cron/scheduler.rs` |
| Cron retry backoff cap | 30s | `cron/scheduler.rs` |
| Cron run output cap | 16,384 bytes | `cron/types.rs` |
| Cron run history | 50 per job | `config/schema.rs` |
| Delegate timeout | 120s / 300s (agentic) | `tools/delegate.rs` |
| Pushover timeout | 15s | `tools/pushover.rs` |
| Channel history | 50 messages | `channels/mod.rs` |
| Channel timeout | 300s base | `channels/mod.rs` |
| Channel timeout scale cap | 4x | `channels/mod.rs` |
| Channel parallelism | 4 per channel | `channels/mod.rs` |
| Max in-flight messages | 8–64 | `channels/mod.rs` |
| Typing indicator refresh | 4s | `channels/mod.rs` |
| Autosave threshold | 20 chars | `channels/mod.rs` |
| Memory context max entries | 4 | `channels/mod.rs` |
| Memory context max chars | 4,000 | `channels/mod.rs` |
| Memory context entry max | 800 chars | `channels/mod.rs` |
| Outbound message cap | 20,000 chars | `channels/mod.rs` |
| History compact keep | 12 turns | `channels/mod.rs` |
| History compact truncate | 600 chars | `channels/mod.rs` |
| Bootstrap max chars | 20,000 (normal) / 6,000 (compact) | `channels/mod.rs` |
| Stream chunk min | 80 chars | `agent/loop_.rs` |
| Max tool iterations | 10 | `agent/loop_.rs` |
| Max history messages | 50 | `agent/loop_.rs` |
| Compaction keep recent | 20 messages | `agent/loop_.rs` |
| Compaction source cap | 12,000 chars | `agent/loop_.rs` |
| Compaction summary cap | 2,000 chars | `agent/loop_.rs` |
| Compaction temperature | 0.2 | `agent/loop_.rs` |
| Progress min interval | 500ms | `agent/loop_.rs` |
| Default temperature | 0.7 | `config/schema.rs` |
| Default actions/hour | 20 | `security/policy.rs` |
| Default cost/day (policy) | 500 cents | `security/policy.rs` |
| Default cost/day (config) | $10.00 | `config/schema.rs` |
| Default cost/month | $100.00 | `config/schema.rs` |
| Cost warn threshold | 80% | `config/schema.rs` |
| Action tracker window | 3,600s (1 hour) | `security/policy.rs` |
| Retry-After cap | 30,000ms | `providers/reliable.rs` |
| Retry base backoff min | 50ms | `providers/reliable.rs` |
| SQLite open timeout cap | 300s | `memory/sqlite.rs` |
| SQLite mmap size | 8 MB | `memory/sqlite.rs` |
| SQLite page cache | ~2 MB | `memory/sqlite.rs` |
| Embedding cache max | 10,000 entries | `config/schema.rs` |
| Embedding dimensions | 1,536 | `config/schema.rs` |
| Vector search weight | 0.7 | `config/schema.rs` |
| Keyword search weight | 0.3 | `config/schema.rs` |
| Min relevance score | 0.4 | `config/schema.rs` |
| Memory recall limit | 5 | `agent/memory_loader.rs` |
| Hygiene archive days | 7 | `config/schema.rs` |
| Hygiene purge days | 30 | `config/schema.rs` |
| Conversation retention | 30 days | `config/schema.rs` |
| Audit log max size | 100 MB | `config/schema.rs` |
| Audit log backups | 10 | `security/audit.rs` |
| OTP token TTL | 30s | `config/schema.rs` |
| OTP cache validity | 300s | `config/schema.rs` |
| Docker memory limit | 512 MB | `config/schema.rs` |
| Docker CPU limit | 1.0 | `config/schema.rs` |
| Hardware baud rate | 115,200 | `config/schema.rs` |
| Computer-use timeout | 15,000ms | `config/schema.rs` |
| Multimodal max images | 4 (clamp: 1–16) | `config/schema.rs` |
| Multimodal max image MB | 5 (clamp: 1–20) | `config/schema.rs` |
| Web search max results | 5 (clamp: 1–10) | `config/schema.rs` |
| Transcription max duration | 120s | `config/schema.rs` |
| Delegate max depth | 3 | `config/schema.rs` |
| Delegate max iterations | 10 | `config/schema.rs` |

---

*End of Technical Blueprint — Files 01 through 05 complete.*
