# 05 — Domain Logic & Technical Debt

## 5.1 Business Rules

### Rule 1: Pairing Code Generation

**Constraint:** Pairing codes must be unambiguous when read aloud or displayed on small screens.

| Parameter | Value | Source |
|---|---|---|
| Length | 8 characters | `PAIRING_CODE_LENGTH` in `src/pairing/pairing-store.ts` |
| Alphabet | `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` (32 chars) | Excludes `0/O/1/I` to prevent visual confusion |
| Max pending per channel | 3 | `PAIRING_PENDING_MAX` |
| TTL | 60 minutes | `PAIRING_PENDING_TTL_MS` |
| Expiry behavior | Pruned on next read (lazy cleanup) | No background sweeper |

**Implication:** If a 4th user contacts the bot before the operator approves any of the first 3, the new request is silently dropped (returns `{ code: "", created: false }`).

---

### Rule 2: Agent Binding Resolution Is Priority-Ordered, Not Weighted

Bindings are evaluated as a strict priority waterfall — the first matching tier wins, and no scoring/weighting is applied.

```
Tier 1: Exact peer match (kind + id)
Tier 2: Parent peer match (thread inherits from channel)
Tier 3: Guild + role match (Discord role-based routing)
Tier 4: Guild match (no role filter)
Tier 5: Team match (Slack team)
Tier 6: Specific account match
Tier 7: Channel wildcard (accountId = "*")
Tier 8: Global default agent
```

**Implication:** A guild-level binding (tier 4) can never override a peer-level binding (tier 1), even if the guild binding was added later. This is by design — specific always wins over general — but it means operators cannot create "exceptions" to peer bindings without removing them.

---

### Rule 3: HEARTBEAT_OK Suppression Has a Character Budget

The agent can reply `HEARTBEAT_OK` (plus up to 300 chars of trailing text) and the response will be suppressed. This prevents unnecessary message delivery.

| Parameter | Value | Source |
|---|---|---|
| Token string | `"HEARTBEAT_OK"` | `src/auto-reply/tokens.ts` |
| Max trailing chars after token strip | 300 | `DEFAULT_HEARTBEAT_ACK_MAX_CHARS` |
| Duplicate suppression window | 24 hours | Heartbeat runner |
| Markup stripping | HTML tags, `&nbsp;`, `*`, backtick, `~`, `_` | `stripHeartbeatToken()` |
| Trailing punctuation strip | Up to 4 non-word chars | `stripTokenAtEdges()` |

**Edge cases:**
- Media payloads bypass text suppression entirely (always delivered)
- Exec completion events bypass suppression (always delivered)
- Suppressed responses trigger transcript pruning (JSONL rolled back to pre-heartbeat offset)

---

### Rule 4: DM Policy + Group Policy Form a Two-Layer Access Matrix

Every inbound message passes through two independent access checks:

| | `dmPolicy: open` | `dmPolicy: pairing` | `dmPolicy: allowlist` | `dmPolicy: disabled` |
|---|---|---|---|---|
| **DM from known sender** | Allow | Allow | Allow | Block |
| **DM from unknown sender** | Allow | Send pairing code, block | Block (silent) | Block |

| | `groupPolicy: open` | `groupPolicy: allowlist` | `groupPolicy: disabled` |
|---|---|---|---|
| **Group, mentioned** | Allow | Allow if group in allowlist | Block |
| **Group, not mentioned** | Block (mention required) | Block | Block |

**Implication:** `dmPolicy: pairing` is the only mode that actively communicates with blocked senders. All other block modes are silent. This is intentional — silent blocking prevents enumeration attacks.

---

### Rule 5: Session DM Scope Controls Context Isolation

The `session.dmScope` config determines whether all DMs share one conversation context or get isolated contexts:

| Scope | Session Key | Behavior |
|---|---|---|
| `main` (default) | `agent:main:main` | All DMs from all senders share one context |
| `per-peer` | `agent:main:direct:<peerId>` | Each sender gets their own context |
| `per-channel-peer` | `agent:main:<channel>:direct:<peerId>` | Isolated per channel + sender |
| `per-account-channel-peer` | `agent:main:<channel>:<account>:direct:<peerId>` | Fully isolated |

**Implication:** With `main` scope (default), user A can see context from user B's conversation if they message the same agent. This is documented behavior for personal-assistant use cases (single operator). Multi-user deployments should use `per-peer` or stricter.

---

### Rule 6: Compaction Uses Token Estimation, Not Real Tokenization

Context window management relies on a `chars ÷ 4` heuristic for token counting, with a 20% safety margin.

| Parameter | Value | Source |
|---|---|---|
| Token estimation | `chars / 4` | Heuristic throughout codebase |
| Safety margin | 1.2x (20% buffer) | `SAFETY_MARGIN` in `src/agents/compaction.ts` |
| Base chunk ratio | 40% of context window | `BASE_CHUNK_RATIO` |
| Min chunk ratio | 15% of context window | `MIN_CHUNK_RATIO` |
| "Too large" threshold | > 50% of context window | Single message overflow guard |
| Summary overhead reserve | 4,096 tokens | `SUMMARIZATION_OVERHEAD_TOKENS` |
| Default context window fallback | 200,000 tokens | `DEFAULT_CONTEXT_TOKENS` for unknown models |

**Implication:** The 20% buffer compensates for multi-byte characters, special tokens, and code tokens being underestimated by the `chars/4` heuristic. In rare cases (heavily CJK or emoji-dense text), this may still be insufficient.

---

### Rule 7: Queue Mode Determines What Happens to Concurrent Messages

When a sender sends multiple messages while an agent turn is already running:

| Mode | Behavior | Use Case |
|---|---|---|
| `none` | Drop (silently discarded) | Strict single-turn |
| `followup` | Enqueue; run after current turn | Default conversational |
| `collect` | Accumulate; run all as single turn | Batch processing |
| `interrupt` | Abort current turn, start new | Urgent-response |
| `steer` | Inject into running agent | Real-time guidance |
| `steer-backlog` | Steer if streaming; followup otherwise | Balanced |
| `steer+backlog` | Steer AND enqueue followup | Comprehensive |
| `queue` | Standard FIFO queue | Ordered processing |

---

### Rule 8: Exec Security Is Three-Tiered

Shell command execution by agents follows a configurable security policy:

| Policy | Behavior |
|---|---|
| `deny` (default) | All exec blocked unless approved per-request |
| `allowlist` | Allowed if command matches allowlist; prompts for unlisted commands |
| `full` | All exec allowed without approval |

**Approval flow parameters:**

| Parameter | Value |
|---|---|
| Default timeout | 120 seconds |
| Grace period (resolved entries) | 15 seconds |
| Decision values | `allow-once`, `allow-always`, `deny` |
| `allow-always` effect | Command added to persistent allowlist |

---

### Rule 9: Channel Restart Has a Backoff Ceiling and Hourly Cap

Channels that disconnect are automatically restarted with exponential backoff:

| Parameter | Value | Source |
|---|---|---|
| Initial delay | 5 seconds | `CHANNEL_RESTART_POLICY.initialMs` |
| Max delay | 5 minutes | `CHANNEL_RESTART_POLICY.maxMs` |
| Backoff factor | 2x | `CHANNEL_RESTART_POLICY.factor` |
| Jitter | 10% | `CHANNEL_RESTART_POLICY.jitter` |
| Max attempts before "gave-up" | 10 | `MAX_RESTART_ATTEMPTS` |
| Health monitor check interval | 5 minutes | `DEFAULT_CHECK_INTERVAL_MS` |
| Startup grace period | 60 seconds | `DEFAULT_STARTUP_GRACE_MS` |
| Max restarts per hour | 3 | `DEFAULT_MAX_RESTARTS_PER_HOUR` |

**Implication:** A channel that exhausts 10 attempts enters "gave-up" state. The health monitor (separate from the restart policy) can reset gave-up channels, but is limited to 3 resets/hour to prevent churn loops.

---

### Rule 10: Cron Error Backoff Escalates to 1-Hour Ceiling

Failed cron jobs use a stepped backoff schedule:

| Consecutive Errors | Backoff Delay |
|---|---|
| 1 | 30 seconds |
| 2 | 1 minute |
| 3 | 5 minutes |
| 4 | 15 minutes |
| 5+ | 60 minutes |

After 3 consecutive **schedule computation errors** (distinct from execution errors), the job is auto-disabled (`enabled = false`). This requires operator intervention to re-enable.

---

### Rule 11: Auth Profile Cooldown Uses Exponential Backoff

When an LLM provider fails (rate limit, billing, auth error), the auth profile enters cooldown:

| Error Count | Cooldown Duration |
|---|---|
| 1 | 5 minutes |
| 2 | 25 minutes |
| 3+ | 60 minutes (cap) |

**Formula:** `min(60 minutes, 5 minutes × 5^(min(errorCount-1, 3)))`

Billing errors (`402`, "insufficient credits") use the same backoff but set a separate `disabledReason: "billing"` flag.

Near cooldown expiry (within 2 minutes), the fallback chain makes probe attempts every 30 seconds to detect recovery early.

---

## 5.2 Validation Rules

### Config Validation

The root config (`openclaw.json`) is validated with Zod schemas on every load. Invalid config blocks gateway startup and surfaces a `doctor` suggestion.

### Message Size Limits (Per Channel)

| Channel | Text Limit | Caption Limit | Other |
|---|---|---|---|
| Telegram | 4,096 chars | 1,024 chars | Max 100 native commands; 64-byte callback data |
| Discord | 2,000 chars | — | 30-min component TTL |
| LINE | 5,000 chars | — | 1 MB webhook body |
| WhatsApp | — | — | 20-min dedup TTL, 5000-entry dedup cache |

### Media Size Limits

| Type | General Limit | Input Processing Limit |
|---|---|---|
| Image | 6 MB | 10 MB |
| Audio | 16 MB | 20 MB (media understanding) |
| Video | 16 MB | 50 MB (media understanding) |
| Document | 100 MB | 5 MB + 200K chars |
| PDF | — | 4 pages max, 4M pixels max |
| Media store | 5 MB | 2-minute TTL |

### Gateway Payload Limits

| Limit | Value |
|---|---|
| WS max payload | 25 MB |
| WS max buffered bytes per connection | 50 MB |
| Chat history cap | 6 MB |
| Handshake timeout | 10 seconds |
| Tick interval | 30 seconds |
| Dedup window | 5 minutes, max 1,000 entries |

### Webhook Limits

| Limit | Value |
|---|---|
| Max body size | 256 KB (configurable) |
| Auth failure lockout | 20 failures per 60 seconds |
| Control-plane writes | 3 per 60 seconds per client |

### Phone Number Validation (Signal)

E.164 format: starts with `+`, 5–15 digits, digits only after prefix.

---

## 5.3 Security Constraints

### SSRF Protection (Dual-Phase)

**Source:** `src/infra/net/ssrf.ts`

1. **Pre-DNS check:** Block known-bad hostnames (`localhost`, `*.localhost`, `*.local`, `*.internal`, `metadata.google.internal`).
2. **Post-DNS check:** After DNS resolution, verify the resolved IP is not in a private range. This prevents public hostnames that resolve to private IPs.
3. **Legacy IPv4 literals** (octal `0177.0.0.1`, hex `0x7f000001`, short-form) are treated as private and blocked.
4. **Malformed IPv6** literals fail closed (treated as private).

Applied to: web fetch tool, browser navigation, media downloads, webhook outbound.

### Sandbox Security Hardening

**Docker container restrictions:**

| Restriction | Enforcement |
|---|---|
| Blocked host mounts | `/etc`, `/proc`, `/sys`, `/dev`, `/root`, `/boot`, `/var/run/docker.sock` |
| Blocked network modes | `host` |
| Blocked security profiles | `unconfined` (seccomp and AppArmor) |
| Default capabilities | `--security-opt no-new-privileges`, capability drops |
| Env var sanitization | Blocks `*API_KEY`, `*TOKEN`, `*PASSWORD`, `*PRIVATE_KEY`, `*SECRET` patterns |
| Large value blocking | Values > 32,768 chars blocked |
| Base64 credential detection | Values matching `^[A-Za-z0-9+/=]{80,}$` trigger warning (not blocked) |

### Timing-Safe Token Comparison

All secret comparisons (gateway token, password, webhook signatures) use:
```
SHA-256(provided) timingSafeEqual SHA-256(expected)
```
Both sides are hashed first to normalize length (prevents length-leaking timing differences).

### Device Auth Replay Protection

- Server-generated nonce per connection (must be signed and returned)
- Timestamp skew limit: ±2 minutes
- Approval IDs bound to `requestedByConnId`, `requestedByDeviceId`, `requestedByClientId` — cross-device replay blocked

### Skill Scanner Security Rules

Agent skills (code loaded from workspace) are scanned for:

| Severity | Rule | Detects |
|---|---|---|
| Critical | `dangerous-exec` | `child_process`, `execSync`, `spawn` |
| Critical | `dynamic-code-execution` | `eval()`, `Function()`, `vm.runIn*` |
| Critical | `crypto-mining` | Mining pool URLs, worker patterns |
| Critical | `env-harvesting` | `process.env` iteration, `Object.keys(process.env)` |
| Warning | `suspicious-network` | Non-standard port connections |
| Warning | `potential-exfiltration` | Data URL construction + fetch patterns |
| Warning | `obfuscated-code` | Long hex/base64 strings, `atob`/`btoa` chains |

---

## 5.4 Edge Cases and Defensive Patterns

### Telegram Text Fragment Buffering

Telegram splits long pastes (~4096 chars) into multiple update events. The system buffers these:

| Parameter | Value |
|---|---|
| Max gap between fragments | 1,500 ms |
| Fragment start threshold | 4,000 chars (only buffers if first message is near limit) |
| Max parts | 12 |
| Max total chars | 50,000 |

Without this buffering, a single user paste would trigger 2–12 separate agent turns.

### Telegram HTML Parse Fallback

If Telegram rejects a message due to HTML entity parsing errors, the system automatically retries with plain text (no formatting). This handles edge cases where the markdown-to-HTML converter produces invalid Telegram HTML.

### Telegram Thread Fallback

If sending to a forum topic fails with "thread not found", the message is retried without the thread ID (sending to the main channel instead). This prevents messages from being lost when topics are deleted. The fallback explicitly does **not** cover "chat not found" errors.

### Session Store Cache Bypass

When resolving session identity for new sessions, the cache is explicitly bypassed (`skipCache: true`). This prevents stale mtime-based cache entries from causing orphaned transcript files when multiple gateway processes or rapid writes are involved (tracked as issue #17971).

### Compaction Race Guard

Session message snapshots are captured with a before/after `isCompacting` check. If compaction was running during the snapshot, the snapshot is discarded (set to `null`) to prevent corrupted compaction input.

### Command Queue Generation Counters

Three independent generation counter systems prevent stale async results:
1. **Telegram draft stream** — prevents superseded preview edits from clobbering newer previews
2. **Heartbeat wake handler** — prevents stale handler disposers from clearing newer handlers after SIGUSR1 restart
3. **Command queue lanes** — prevents stale task completions from resolving after `cancelLane()`

### Cron Daily Schedule Jump Prevention

A specific fix for issue #17852: between `findDueJobs` and the locked execution block, a full `recomputeNextRuns` would silently advance `nextRunAtMs` past due jobs without executing them, causing daily cron schedules to jump 48h instead of 24h. The fix uses `recomputeNextRunsForMaintenance` which only updates maintenance fields.

---

## 5.5 Technical Debt

### Debt 1: Token Counting Is a Heuristic, Not Exact

The entire compaction and context-window system relies on `chars / 4` for token estimation. A 20% safety margin compensates, but this is inherently fragile for:
- CJK text (typically 1–2 chars per token, not 4)
- Code with many special tokens
- Repeated tokens (tokenizer-specific compression)

**Risk:** Context overflow errors on edge-case inputs despite the safety margin.
**Mitigation path:** Integrate a real tokenizer (e.g., `tiktoken` for OpenAI models, `@anthropic-ai/tokenizer` for Claude).

---

### Debt 2: Inconsistent Media Size Limits Across Modules

Three separate files define overlapping media size limits:

| File | Constant | Value |
|---|---|---|
| `src/media/constants.ts` | `MAX_IMAGE_BYTES` | 6 MB |
| `src/media/input-files.ts` | `DEFAULT_INPUT_IMAGE_MAX_BYTES` | 10 MB |
| `src/media/store.ts` | `MEDIA_MAX_BYTES` | 5 MB |

These are used in different contexts (outbound, inbound processing, media store) but the naming doesn't make the scoping clear. A unified media limits config would reduce confusion.

---

### Debt 3: Silent Constant Coupling Between Files

`MAX_RESTART_ATTEMPTS = 10` in `src/gateway/server-channels.ts` is referenced as a hardcoded `10` in `src/gateway/channel-health-monitor.ts` (line 127). Changing one without the other would create a behavior gap where the health monitor misclassifies channel states.

---

### Debt 4: Control-Plane Rate Limiter Leaks Memory

`controlPlaneBuckets` in `src/gateway/control-plane-rate-limit.ts` is a `Map` with no periodic cleanup. Unlike the auth rate limiter (which has explicit pruning every 60s), old entries from disconnected devices accumulate indefinitely. For long-running gateways with many device connections, this is a slow memory leak.

---

### Debt 5: Legacy Field Migrations Run on Every Load

Several migrations run on every config/store load rather than as one-time upgrades:
- Cron `payload.provider` → `payload.channel` rename
- Cron `schedule.atMs` → `schedule.at` rename
- Pairing allowFrom legacy-to-scoped path merge
- Auth store `auth.json` → `auth-profiles.json`

These are idempotent and fast, but they add read-path overhead and will accumulate as the schema evolves.

---

### Debt 6: Session Transcript Integrity Is Convention-Enforced

The session transcript (JSONL) uses a `parentId` DAG structure for message ordering. Writing raw JSONL entries without `parentId` severs the leaf path, breaking compaction and history traversal. This constraint is documented in `src/gateway/server-methods/CLAUDE.md` rather than enforced at the type system level.

**Risk:** Any code path that appends to JSONL directly (bypassing `SessionManager.appendMessage()`) could corrupt session state.

---

### Debt 7: WebSocket Dedup Cache Has No Persistence

The WhatsApp inbound dedup cache (`src/web/inbound/dedupe.ts`) is in-memory only with a 20-minute TTL and 5,000-entry cap. A gateway restart clears the cache, potentially allowing re-delivery of messages received in the last 20 minutes.

---

### Debt 8: Node.js 22 IPv6/DNS Workarounds

Two workarounds in `src/telegram/fetch.ts` force IPv4-first DNS resolution and auto-select-family behavior to handle Node 22 regressions with Telegram's API servers. These reference specific Node.js issues (#54359, internal #5311) and should be revisited when Node 22 DNS behavior stabilizes.

---

### Debt 9: QMD Multi-Collection Query Workaround

The QMD memory backend doesn't support multi-collection queries natively. The code runs one query per collection and merges results with deduplication. This is explicitly logged as a "workaround" and adds latency proportional to the number of collections.

---

## 5.6 Platform-Specific Workarounds

| Platform | Workaround | Reason |
|---|---|---|
| Telegram | HTML parse fallback to plain text | Telegram's HTML parser rejects some valid constructs |
| Telegram | Thread-not-found fallback to main channel | Deleted forum topics cause send failures |
| Telegram | Text fragment buffering (12 parts / 50K chars) | Platform splits large pastes into multiple updates |
| Telegram | IPv4-first DNS + auto-select-family | Node 22 IPv6 regressions with Telegram API |
| WhatsApp | In-memory dedup cache (20min / 5000 entries) | Baileys may deliver duplicate messages |
| Discord | 30-minute component TTL | Discord interaction tokens expire |
| LINE | Signature verification with HMAC-SHA256 | LINE webhook security requirement |
| Windows | Session store atomic write retry (5 attempts with backoff) | Windows `rename()` can fail under concurrent access |
| Feishu | ID prefix-based chat type inference (`oc_`/`ou_`/`on_`) | Feishu API doesn't expose chat type in all contexts |
