# 05 — Domain Logic & Technical Debt

## 5.1 Security Rules

IronClaw enforces defense-in-depth through four layered systems. Every piece of external data passes through multiple checks before reaching the LLM or the user.

### Safety Pipeline Order

```
External Input
  │
  ├─1─► Validator    (length, encoding, forbidden patterns)
  ├─2─► Sanitizer    (injection detection, escaping)
  ├─3─► Policy       (rule matching → block/warn/sanitize/review)
  └─4─► Leak Detector (secret scanning → block/redact/warn)
  │
  ▼
Safe Output
```

---

### 5.1.1 Input Validation Rules

| Rule | Threshold | Action |
|------|-----------|--------|
| Max input length | 100,000 bytes | Reject (`TooLong`) |
| Min input length | 1 byte | Reject (`TooShort`) |
| Null bytes | Any `\x00` | Reject (`InvalidEncoding`) |
| Whitespace ratio | >90% whitespace AND >100 chars | Warning |
| Excessive repetition | >20 repeated chars in strings ≥50 chars | Warning |

### 5.1.2 Prompt Injection Detection

The sanitizer uses Aho-Corasick multi-pattern matching (fast, case-insensitive) plus regex fallbacks:

**Critical severity (blocked):**
- `"ignore all previous"` — instruction override
- `"system:"`, `"[INST]"`, `"[/INST]"` — system message injection
- `"<|"`, `"|>"` — special token injection (model control tokens)
- Null bytes (`\x00`)

**High severity (sanitized):**
- `"ignore previous"`, `"forget everything"`, `"new instructions"`, `"updated instructions"` — instruction override
- `"you are now"`, `"assistant:"`, `"user:"` — role manipulation
- `eval(`, `exec(` — code injection
- `` ```system `` — code block system injection

**Medium severity (warned):**
- `"act as"`, `"pretend to be"`, `"disregard"` — mild role manipulation
- Base64 payloads (>50 chars)
- `` ```bash\nsudo `` — privilege escalation in code blocks

**Escaping logic:** When content is sanitized rather than blocked, dangerous tokens are neutralized:
- `<|` → `\<|`, `|>` → `\|>`
- `[INST]` → `\[INST]`, `[/INST]` → `\[/INST]`
- Lines starting with `system:`, `user:`, `assistant:` are prefixed with `[ESCAPED]`
- Null bytes removed entirely

### 5.1.3 Policy Rules

8 default rules evaluated against all content:

| Rule | Pattern | Severity | Action |
|------|---------|----------|--------|
| System file access | `/etc/passwd`, `/etc/shadow`, `.ssh/`, `.aws/credentials` | Critical | Block |
| Crypto private key | private key + 64-char hex | Critical | Block |
| Shell injection | `; rm -rf`, `; curl ... | sh` | Critical | Block |
| Encoded exploit | `base64_decode`, `eval(base64`, `atob(` | High | Sanitize |
| SQL patterns | `DROP TABLE`, `DELETE FROM`, `INSERT INTO`, `UPDATE...SET` | Medium | Warn |
| Excessive URLs | 10+ URLs in content | Low | Warn |
| Obfuscated string | 500+ chars without whitespace | Medium | Warn |

**Policy actions:**
- **Block** — reject entirely, return error
- **Sanitize** — escape dangerous content, continue processing
- **Warn** — log warning, allow through
- **Review** — flag for manual review (not auto-resolved)

### 5.1.4 Secret Leak Detection

16+ regex patterns scanned at two points: (1) tool output before it reaches the LLM, and (2) LLM responses before they reach the user.

**Critical / Block:**

| Pattern | Regex | Example |
|---------|-------|---------|
| OpenAI API key | `sk-(?:proj-)?[a-zA-Z0-9]{20,}` | `sk-proj-abc123...` |
| Anthropic API key | `sk-ant-api[a-zA-Z0-9_-]{90,}` | `sk-ant-api...` (90+ chars) |
| AWS access key | `AKIA[0-9A-Z]{16}` | `AKIAIOSFODNN7EXAMPLE` |
| GitHub token | `gh[pousr]_[A-Za-z0-9_]{36,}` | `ghp_...` (36+ chars) |
| GitHub fine-grained PAT | `github_pat_[a-zA-Z0-9]{22}_[a-zA-Z0-9]{59}` | Specific format |
| Stripe API key | `sk_(?:live\|test)_[a-zA-Z0-9]{24,}` | `sk_live_...` |
| NEAR AI session | `sess_[a-zA-Z0-9]{32,}` | `sess_...` (32+ chars) |
| PEM private key | `-----BEGIN.*PRIVATE KEY-----` | RSA, EC, generic |
| SSH private key | `-----BEGIN OPENSSH PRIVATE KEY-----` | OpenSSH, EC, DSA |

**High / Block:**
Google API key (`AIza...`), Slack token (`xoxb-...`), Twilio key (`SK` + 32 hex), SendGrid key (`SG....`)

**High / Redact:**
Bearer tokens (`Bearer [20+ chars]`), Authorization headers

**Medium / Warn:**
High-entropy hex strings (64+ hex chars)

**Masking format:** First 4 + `****` + last 4 chars (e.g., `sk-t****cdef`)

---

## 5.2 Shell Execution Rules

The shell tool has the most complex security model in the system.

### Blocked Commands (exact match, always rejected)

```
rm -rf /          rm -rf /*         :(){ :|:& };:     (fork bomb)
dd if=/dev/zero   mkfs              chmod -R 777 /
> /dev/sda        curl | sh         wget | sh
curl | bash       wget | bash
```

### Dangerous Patterns (substring match, rejected)

```
sudo              doas              | sh
| bash            | zsh             eval
$(curl            $(wget            /etc/passwd
/etc/shadow       ~/.ssh            .bash_history
id_rsa
```

### Never Auto-Approve Patterns (42 patterns)

Even if the user has set a tool to "always approve", these patterns still require explicit per-execution approval:

**Destructive:** `rm -rf`, `rm -fr`, `chmod 777`, `chown -r`
**System:** `shutdown`, `reboot`, `poweroff`, `init 0/6`, `iptables`, `nft`
**User mgmt:** `useradd`, `userdel`, `passwd`, `visudo`, `crontab`
**Services:** `systemctl disable`, `launchctl unload`
**Process:** `kill -9`, `killall`, `pkill`
**Docker:** `docker rm`, `docker rmi`, `docker system prune`
**Git (destructive):** `git push --force`, `git push -f`, `git reset --hard`, `git clean -f`
**Database:** `DROP TABLE`, `DROP DATABASE`, `TRUNCATE`, `DELETE FROM`

### Command Injection Detection (7 heuristics)

1. Null bytes in command string
2. Base64 decode piped to shell (`base64 -d | sh`)
3. Printf/echo encoded piped to shell (`printf '\x..' | sh`)
4. Hex reverse piped to shell (`xxd -r | sh`)
5. DNS exfiltration via command substitution (`dig $(cmd)`)
6. Netcat-based data piping (`nc evil.com | ...`)
7. Curl/wget file exfiltration (`curl -d @file`, `curl --upload-file`)

### Environment Scrubbing

Only 32 safe environment variables are forwarded to child processes. All others (including `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `DATABASE_URL`, etc.) are scrubbed. Safe list includes: `PATH`, `HOME`, `USER`, `SHELL`, `TERM`, `LANG`, `LC_*`, `PWD`, `TMPDIR`, `CARGO_HOME`, `RUSTUP_HOME`, `NODE_PATH`, `EDITOR`, `VISUAL`, plus Windows equivalents.

---

## 5.3 Sandbox & WASM Constraints

### Docker Sandbox Resource Limits

| Resource | Default | Configurable |
|----------|---------|-------------|
| Memory | 2,048 MB | `sandbox.memory_limit_mb` |
| CPU shares | 1,024 | `sandbox.cpu_shares` |
| Timeout | 120 seconds | `sandbox.timeout_secs` |
| Max output | 64 KB | Hardcoded |
| Execution user | UID 1000 (non-root) | No |
| Root filesystem | Read-only | No |
| Linux capabilities | All dropped | No |
| Container cleanup | Auto (--rm + explicit) | No |

### WASM Sandbox Resource Limits

| Resource | Default | Configurable |
|----------|---------|-------------|
| Memory | 10 MB | `wasm.default_memory_limit` |
| Fuel (instructions) | 10,000,000 | `wasm.default_fuel_limit` |
| Timeout | 60 seconds | `wasm.default_timeout_secs` |
| Max tables | 10 | Hardcoded |
| Max instances | 10 | Hardcoded |
| Max table growth | 10,000 entries | Hardcoded |

### WASM Network Allowlist Rules

- HTTPS required by default (HTTP opt-in only)
- Userinfo in URLs rejected (`user:pass@host` — prevents allowlist confusion)
- Path traversal blocked (normalized `..` and `%.` sequences)
- Invalid percent-encoding rejected
- Encoded path separators blocked (`%2F` in segments)

### WASM Channel Limits

| Limit | Value |
|-------|-------|
| Min poll interval | 30,000 ms (30 seconds) |
| Default emit rate | 100 per minute |
| Default emit rate | 5,000 per hour |
| Max emits per execution | 100 |
| Max message content | 64 KB |
| Max log message | 4,096 bytes |
| Max HTTP requests per execution | 50 |
| Max invoke calls per execution | 20 |

---

## 5.4 Rate Limits & Capacity Rules

| Scope | Limit | Window |
|-------|-------|--------|
| Chat messages (web API) | 30 | 60 seconds (sliding) |
| Shell commands | 30 | 300 seconds |
| Memory writes | 20/min, 200/hour | Sliding |
| Job creation | 5/min, 30/hour | Sliding |
| Parallel jobs | 5 concurrent | — |
| HTTP webhook channel | 60 requests/min | Sliding |
| HTTP webhook body | 64 KB max | — |
| HTTP webhook content | 32 KB max | — |
| HTTP webhook pending | 100 responses | Queue |
| WASM tool (default) | 60/min, 1,000/hour | Per-tool, per-user |
| WASM HTTP request body | 1 MB max | Per-tool |
| WASM HTTP response body | 10 MB max | Per-tool |
| WASM HTTP timeout | 30 seconds | Per-tool |
| Max SSE connections | 100 concurrent | — |
| Extension download | 50 MB max | Per-download |
| Extension capabilities file | 1 MB max | Per-file |
| Extension archive entry | 100 MB max | Per-entry |

---

## 5.5 Agent Behavior Rules

### Iteration & Completion

| Rule | Value | Purpose |
|------|-------|---------|
| Hard iteration ceiling | 500 | Absolute maximum per job (prevents runaway) |
| Default soft limit | 50 per job | Configurable via `agent.max_tool_iterations` |
| Nudge threshold | limit - 1 | Inject "wrap up" hint to LLM |
| Force text threshold | limit | Force LLM to respond with text, no tools |
| Completion signal detection | Word-boundary matching | Prevents false positives on "completely" |
| Negation blocking | "not complete", "incomplete" | Prevents premature completion |

### Self-Repair

| Rule | Value |
|------|-------|
| Stuck detection threshold | 5 minutes without progress |
| Max repair attempts per job | 3 (configurable) |
| Broken tool detection threshold | 5 cumulative failures |
| Repair check interval | 60 seconds |
| Session pruning interval | 600 seconds (10 min) |
| Session idle timeout | 604,800 seconds (7 days) |
| Session count warning | ≥1,000 active sessions |

### Context Management

| Rule | Value |
|------|-------|
| Default context limit | 100,000 tokens |
| Compaction trigger | 80% of limit |
| Token estimation | 1.3 tokens per word + 4 per message |
| Critical pressure (>95%) | Truncate to 3 recent turns |
| High pressure (>85%) | Summarize, keep 5 turns |
| Moderate pressure (70-85%) | Move to workspace, keep 10 turns |
| Summary generation | max_tokens=1024, temperature=0.3 |

### Estimation Learning

| Parameter | Value |
|-----------|-------|
| EMA smoothing factor (alpha) | 0.1 |
| Min samples before adjustment | 5 |
| Alpha range | [0.01, 0.5] |
| Confidence formula | `0.5 + (min(samples/100, 1) × 0.3) + ((1 - avg_error) × 0.2)` |

---

## 5.6 Credential Security Model

**Zero-exposure architecture:** Secrets never enter container processes.

```
┌──────────────┐     ┌────────────────┐     ┌──────────────┐
│ Secret Store │────►│  Host Proxy    │────►│  Container   │
│ (AES-256-GCM │     │  (decrypts,    │     │  (sees only  │
│  encrypted)  │     │   injects into │     │   HTTP with  │
│              │     │   HTTP headers)│     │   auth added)│
└──────────────┘     └────────────────┘     └──────────────┘
                     Never leaves host       Never holds secret
```

**Four injection methods:**
1. **Bearer:** `Authorization: Bearer {secret}`
2. **Basic:** `Authorization: Basic {base64(user:secret)}`
3. **Custom header:** `X-Header-Name: {optional_prefix}{secret}`
4. **Query parameter:** `?param_name={secret}`

**Host pattern matching:** Exact (`api.example.com`) or wildcard (`*.example.com`). Empty prefix does NOT match wildcard.

**Default credential mappings (3):**

| Env Var | Domain | Method |
|---------|--------|--------|
| `OPENAI_API_KEY` | `api.openai.com` | Bearer |
| `ANTHROPIC_API_KEY` | `api.anthropic.com` | `x-api-key` header |
| `NEARAI_API_KEY` | `api.near.ai` | Bearer |

---

## 5.7 Skill Trust Enforcement

**Critical business rule:** If ANY installed (registry-sourced) skill is active in the current LLM context, the tool ceiling drops to the read-only subset for ALL skills in that turn — even trusted ones.

**Read-only tool subset:** `memory_search`, `memory_read`, `memory_tree`, `time`, `echo`, `json`, `skill_list`, `skill_search`

**Rationale:** A malicious skill from the registry could inject instructions into the LLM prompt that abuse trusted tools (shell, file write, HTTP). The attenuation ensures that even if the LLM follows injected instructions, it cannot access dangerous tools.

**Skill constraints:**

| Limit | Value |
|-------|-------|
| Max keywords per skill | 20 |
| Max patterns per skill | 5 |
| Max tags per skill | 10 |
| Min keyword/tag length | 3 characters |
| Max SKILL.md file size | 64 KB |
| Max regex pattern size | 64 KB |
| Max context tokens (all skills) | 4,000 |
| Catalog cache TTL | 300 seconds |
| Catalog max results | 25 |
| Catalog request timeout | 10 seconds |

---

## 5.8 Network Allowlist (Default)

Containers can only reach these domains (configurable via `sandbox.extra_allowed_domains`):

**Package registries:**
`crates.io`, `static.crates.io`, `index.crates.io`, `registry.npmjs.org`, `proxy.golang.org`, `pypi.org`, `files.pythonhosted.org`

**Documentation:**
`docs.rs`, `doc.rust-lang.org`, `nodejs.org`, `go.dev`, `docs.python.org`

**Version control:**
`github.com`, `raw.githubusercontent.com`, `api.github.com`, `codeload.github.com`

**LLM APIs:**
`api.openai.com`, `api.anthropic.com`, `api.near.ai`

---

## 5.9 Technical Debt & Known Limitations

### Incomplete Implementations

| Area | Issue | Location |
|------|-------|----------|
| WIT bindgen | Tool description and schema extracted via stubs, not WIT bindgen | `src/tools/wasm/runtime.rs:273,286` |
| Self-repair timing | Only attempt-count based; time-based stuck detection not wired | `src/agent/self_repair.rs:69` |
| Tool hot-reload | After repair, tool not hot-reloaded into registry | `src/agent/self_repair.rs:75` |
| OAuth state nonce | CSRF protection via state nonce not implemented for WASM channel OAuth | `src/channels/wasm/router.rs:446` |
| Webhook credential substitution | Secret substitution into webhook validation URLs not implemented | `src/setup/channels.rs:774` |
| MCP stdio transport | Only HTTP transport implemented; stdio (most common) missing | CLAUDE.md |
| Webhook trigger endpoint | Routine webhook triggers not exposed in web gateway | CLAUDE.md |
| Per-channel status dashboard | Gateway status widget exists but no per-channel connection view | CLAUDE.md |

### libSQL Backend Gaps

| Gap | Impact |
|-----|--------|
| No encryption at rest | SQLite stores all data in plaintext (only secrets are AES-encrypted) |
| Workspace not wired through Database trait | Memory system requires Store migration |
| Secrets store not available | Still requires PostgresSecretsStore |
| Vector search not implemented | FTS5 only; `libsql_vector_idx` not wired |
| Settings reload from DB skipped | Config::from_db requires Store |
| No incremental migrations | Schema is CREATE IF NOT EXISTS, no ALTER TABLE |
| JSON merge patch semantics | `json_patch` (RFC 7396) replaces top-level keys entirely vs PostgreSQL's path-targeted `jsonb_set`; callers must avoid partial nested updates |

### Stub Implementations

| Module | Status |
|--------|--------|
| `src/tools/builtin/marketplace.rs` | Returns placeholder responses |
| `src/tools/builtin/restaurant.rs` | Returns placeholder responses |
| `src/tools/builtin/taskrabbit.rs` | Returns placeholder responses |
| `src/tools/builtin/ecommerce.rs` | Returns placeholder responses |

### Hardcoded Values That Should Be Configurable

| Value | Location | Current | Should Be |
|-------|----------|---------|-----------|
| Max SSE connections | `src/channels/web/sse.rs` | 100 | Config/env var |
| SSE broadcast buffer | `src/channels/web/sse.rs` | 256 | Config |
| WebSocket direct channel buffer | `src/channels/web/ws.rs` | 64 | Config |
| Worker message channel buffer | `src/agent/scheduler.rs` | 16 | Config |
| Max output size (shell) | `src/tools/builtin/shell.rs` | 64 KB | Config |
| Max file read size | `src/tools/builtin/file.rs` | 1 MB | Config |
| Max file write size | `src/tools/builtin/file.rs` | 5 MB | Config |
| Max directory entries | `src/tools/builtin/file.rs` | 500 | Config |
| Hard iteration ceiling | `src/agent/worker.rs` | 500 | Config (safety cap) |
| Broken tool threshold | `src/agent/self_repair.rs` | 5 failures | Config |
| Summary temperature | `src/agent/compaction.rs` | 0.3 | Config |
| Tokens per word estimate | `src/agent/context_monitor.rs` | 1.3 | Config per model |
| LLM cache max entries | `src/llm/response_cache.rs` | 1,000 | Config |
| LLM cache TTL | `src/llm/response_cache.rs` | 1 hour | Config |
| Circuit breaker failure threshold | `src/llm/circuit_breaker.rs` | 5 | Config |
| Circuit breaker recovery timeout | `src/llm/circuit_breaker.rs` | 30s | Config |
| Retry max attempts | `src/llm/retry.rs` | 3 | Config |
| EMA smoothing alpha | `src/estimation/learner.rs` | 0.1 | Config |

### Architecture Observations

1. **No external queue** — Job scheduling uses in-process `HashMap` + `RwLock`. If the process crashes, all in-flight jobs are lost. No persistence of job queue state between restarts.

2. **Single-process architecture** — The agent, web gateway, orchestrator, and routine engine all run in one process. No horizontal scaling path exists.

3. **Token estimation is approximate** — Context monitoring uses `words × 1.3 + 4` which varies significantly by model and language. Could lead to premature or late compaction.

4. **Tool output size uncapped for non-shell tools** — While shell output is capped at 64 KB, other tools (HTTP, file read) have larger limits (5 MB, 1 MB) that could flood LLM context.

5. **Rate limiter is in-memory** — WASM tool rate limiting state is in the database, but the web gateway chat rate limiter is in-memory and resets on restart.

6. **Skill attenuation is all-or-nothing** — A single installed skill drops ALL tool access to read-only, even for co-active trusted skills. This is secure but may be overly restrictive for common use cases.
