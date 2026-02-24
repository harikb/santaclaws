# 05 — Domain Logic & Technical Debt

> Business rules extracted from code, validation constraints, security boundaries, edge cases, and identified debt.

---

## 1. Business Rules

### 1.1 Agent Execution Rules

| Rule | Constraint | Source |
|---|---|---|
| **Tool iteration cap** | An agent may call at most `max_tool_iterations` tools per conversation turn (default: 20). After this limit, the loop exits and delivers whatever content is available. | `pkg/agent/loop.go` |
| **Context overflow retry** | If the LLM returns a token/context/length error, the system retries up to **2 times**. On the first retry, it notifies the user ("Context window exceeded...") and calls `forceCompression()`. | `pkg/agent/loop.go` |
| **Force compression** | Emergency compression drops the **oldest 50%** of conversation messages, keeping the system prompt and the last 4 messages. A system note is injected: `"[System Note: Emergency compression dropped N oldest messages due to context limit]"` | `pkg/agent/loop.go` |
| **Summarization trigger** | Summarization is triggered when session history exceeds **20 messages** OR estimated tokens exceed **75% of the context window**. | `pkg/agent/loop.go` |
| **Summarization batch split** | If more than **10 messages** need summarizing, they are split in half and summarized in two passes, then merged. Messages exceeding `context_window / 2` tokens are skipped. | `pkg/agent/loop.go` |
| **Summarization output cap** | Summary LLM calls use `max_tokens: 1024` and a **120-second timeout**. | `pkg/agent/loop.go` |
| **Token estimation** | Tokens are estimated as `total_chars × 2 / 5` (~2.5 chars per token). This is a rough heuristic that works reasonably for English and CJK text. | `pkg/agent/loop.go` |
| **Empty response substitution** | If the LLM produces an empty response, the system substitutes: `"I've completed processing but have no response to give."` | `pkg/agent/loop.go` |
| **Deduplication guard** | If a tool already sent a message to the user during the turn (tracked via `HasSentInRound()`), the agent loop does **not** send the final response again. | `pkg/agent/loop.go` |
| **Internal channels skip state recording** | Messages from `cli`, `system`, or `subagent` channels are not recorded as last-used channel/chat. | `pkg/agent/loop.go` |

### 1.2 Routing Rules

| Rule | Constraint | Source |
|---|---|---|
| **Implicit default agent** | If no agents are configured, the system creates an agent with `id: "main"` using all defaults. | `pkg/agent/registry.go` |
| **7-level binding cascade** | Bindings are evaluated in strict priority: peer → parent peer → guild → team → account → channel wildcard → default. First match wins. | `pkg/routing/route.go` |
| **Agent ID normalization** | IDs are lowercased, invalid characters replaced with `-`, leading/trailing dashes stripped. Max **64 characters**. Empty input defaults to `"main"`. Pattern: `^[a-z0-9][a-z0-9_-]{0,63}$`. | `pkg/routing/agent_id.go` |
| **Account ID default** | Empty account IDs default to `"default"`. | `pkg/routing/agent_id.go` |
| **DM session scoping** | `main` = all DMs share one session; `per-peer` = per user; `per-channel-peer` = per channel+user; `per-account-channel-peer` = full isolation. Groups always include group ID. | `pkg/routing/session_key.go` |

### 1.3 Channel Access Rules

| Rule | Constraint | Source |
|---|---|---|
| **Allow-list enforcement** | If a channel's `allow_from` list is non-empty, only listed sender IDs may interact. Unlisted senders are silently dropped — no error response. | `pkg/channels/base.go` |
| **Flexible ID types** | `allow_from` accepts both strings and numbers in JSON (e.g., `[123, "username"]`). The `FlexibleStringSlice` type handles both. | `pkg/config/config.go` |
| **Discord mention-only mode** | When `mention_only: true`, the bot only responds in guild channels when @-mentioned. DMs always get responses. | `pkg/channels/discord.go` |
| **OneBot group trigger prefix** | In group chats, messages must start with a configured prefix OR @-mention the bot. Empty prefix strings are skipped. | `pkg/channels/onebot.go` |

### 1.4 Skill Rules

| Rule | Constraint | Source |
|---|---|---|
| **Skill name validation** | Pattern: `^[a-zA-Z0-9]+(-[a-zA-Z0-9]+)*$`. Length: 1–64 chars. | `pkg/skills/installer.go` |
| **Skill description limit** | 1–1024 characters. | `pkg/skills/installer.go` |
| **Path traversal block** | Skill identifiers containing `/`, `\`, or `..` are rejected. | `pkg/utils/skills.go` |
| **Three-tier override** | Workspace skills (tier 1) override global skills (tier 2) which override built-in skills (tier 3). Same-name skills in a higher tier completely shadow lower tiers. | `pkg/skills/loader.go` |
| **Malware hard-block** | If `is_malware_blocked = true` in metadata, the skill is removed after download and installation fails with an error. | `pkg/tools/skills_install.go` |
| **Suspicious soft-warn** | If `is_suspicious = true`, installation succeeds but a warning is prepended to the result. | `pkg/tools/skills_install.go` |
| **Concurrent install lock** | A workspace-level mutex prevents concurrent skill installations. | `pkg/tools/skills_install.go` |
| **Force reinstall** | `force: true` removes the existing skill directory before downloading. Without it, attempting to install an existing skill returns an error. | `pkg/tools/skills_install.go` |

### 1.5 Cron Rules

| Rule | Constraint | Source |
|---|---|---|
| **Exec timeout** | Cron task execution is capped at `exec_timeout_minutes` (default: **5 minutes**). | `pkg/config/defaults.go` |
| **One-time auto-delete** | Jobs with `deleteAfterRun: true` are removed from the store after execution. | `pkg/cron/service.go` |
| **Command delivery override** | When `command` is set on a cron payload, `deliver` is forced to `false`. Commands are always processed by the agent, never sent raw to a channel. | `pkg/tools/cron.go` |

### 1.6 Subagent Rules

| Rule | Constraint | Source |
|---|---|---|
| **Spawn permission** | A parent agent can only spawn agents listed in its `subagents.allow_agents`. `["*"]` = any agent. `[]` or omitted = no spawning. | `pkg/agent/registry.go` |
| **Spawn is async** | The `spawn` tool returns immediately. The subagent runs in a background goroutine. Completion is reported via a `"system"` channel inbound message. | `pkg/tools/spawn.go` |
| **Subagent is sync** | The `subagent` tool blocks until the child completes all tool iterations. Returns dual output (brief for user, full for LLM). | `pkg/tools/subagent.go` |

---

## 2. Validation Rules

### 2.1 Input Validation

| Field | Rule | Error Behavior |
|---|---|---|
| Agent ID | `^[a-z0-9][a-z0-9_-]{0,63}$` after normalization | Invalid chars replaced with `-`; silently corrected, never rejected |
| Skill slug | `^[a-zA-Z0-9]+(-[a-zA-Z0-9]+)*$`, 1–64 chars, no `/\..` | Rejected with error message |
| Skill description | 1–1024 chars | Rejected with error message |
| Shell command | Matched against 42 deny-pattern regexes (when enabled) | Blocked with error |
| File path (workspace-restricted) | Must resolve within agent workspace | Blocked with error |
| `edit_file` old_text | Must appear exactly once in file | Rejected if not found or ambiguous |
| `web_fetch` URL | Must be `http://` or `https://` | Rejected with error |
| I2C address | `0x03`–`0x77` (7-bit range) | Rejected if out of range |
| I2C/SPI write | Requires `confirm: true` | Rejected without confirmation |
| SPI speed | 1 Hz – 125 MHz | Clamped or rejected |
| SPI mode | 0–3 | Rejected if out of range |
| Cron `at_seconds` | Must be positive integer | Rejected |
| Search `limit` | Clamped to [1, 20] | Silently adjusted |
| Web search `count` | Clamped to [1, 10] | Silently adjusted |

### 2.2 Session Filename Validation

| Check | Purpose |
|---|---|
| `:` replaced with `_` | Avoids Windows volume separator issues |
| `filepath.IsLocal()` check | Blocks `..`, absolute paths, device names |
| No directory separators in sanitized key | Prevents writing outside session directory |

### 2.3 History Sanitization (Before LLM Call)

| Rule | Action |
|---|---|
| Tool message without preceding assistant with tool_calls | Dropped (orphaned result) |
| Assistant with tool_calls at start of history or after non-user/tool message | Dropped (invalid sequence) |
| Consecutive system messages | Prevented during compression (Zhipu API compatibility) |

---

## 3. Security Boundaries

### 3.1 Shell Command Sandbox

**42 deny patterns** guard against destructive or dangerous commands when `enable_deny_patterns = true`:

| Category | Patterns Blocked |
|---|---|
| **Filesystem destruction** | `rm -rf`, `del /f`, `rmdir /s`, `format`, `mkfs`, `diskpart`, `dd if=`, `> /dev/sd*` |
| **System control** | `shutdown`, `reboot`, `poweroff`, fork bombs |
| **Privilege escalation** | `sudo`, `chmod`, `chown` |
| **Process killing** | `pkill`, `killall`, `kill -9` |
| **Command injection** | `$()`, `${}`, backticks, `| sh`, `| bash`, `eval`, `source *.sh`, heredocs |
| **Chained destruction** | `; rm -rf`, `&& rm -rf`, `|| rm -rf` |
| **Network fetch-to-exec** | `curl | sh`, `wget | sh` |
| **Package managers** | `npm install -g`, `pip install --user`, `apt install/remove`, `yum`, `dnf` |
| **Container escape** | `docker run`, `docker exec` |
| **VCS push** | `git push`, `git force` |
| **Remote access** | `ssh *@*` |
| **Dynamic eval** | `eval`, `source *.sh`, `$(cat/curl/wget/which ...)` |

**Default: DISABLED.** The deny patterns are opt-in via `tools.exec.enable_deny_patterns`. This means the default configuration allows unrestricted shell execution within the workspace.

**Timeout:** Commands are killed after **60 seconds** (2-second SIGTERM grace period before SIGKILL).

**Output truncation:** Stdout/stderr capped at **10,000 characters** to prevent memory exhaustion.

### 3.2 Workspace Isolation

When `restrict_to_workspace = true` (default):
- `read_file`, `write_file`, `append_file`, `list_dir`, `edit_file` — path must resolve within workspace
- `exec` — working directory must be within workspace
- Path traversal (`../`) is detected and rejected

### 3.3 Webhook Verification

| Channel | Mechanism | Detail |
|---|---|---|
| LINE | HMAC-SHA256 | Request body signed with channel secret, verified via `X-Line-Signature` header |
| WeCom Bot | SHA1 + AES-CBC | Signature: `SHA1(sort(token, timestamp, nonce, msg_encrypt))`. Decryption: AES-CBC with PKCS7 padding (block size 32). IV = first 16 bytes of AES key. Decrypted payload validated for `receiveid` match. |
| WeCom App | SHA1 + AES-CBC + Corp ID | Same as WeCom Bot, plus `corp_id` verified in decrypted payload |

### 3.4 Token/Secret Handling

- API keys stored in `config.json` (plaintext) or environment variables
- OAuth tokens stored in `~/.picoclaw/auth/tokens.json` (plaintext)
- No encryption at rest for credentials
- ClawHub auth token sent as `Authorization: Bearer {token}` header

### 3.5 Download Safety

| Guard | Value | Source |
|---|---|---|
| Max ZIP download size | 50 MB | `pkg/skills/clawhub_registry.go` |
| Max API response size | 2 MB | `pkg/skills/clawhub_registry.go` |
| Malware flag check | Hard-block + cleanup | `pkg/tools/skills_install.go` |
| Skill identifier validation | No path traversal chars | `pkg/utils/skills.go` |

---

## 4. Edge Cases & Platform-Specific Behavior

### 4.1 Platform Branching

| Behavior | Unix | Windows |
|---|---|---|
| Shell execution | `sh -c "{command}"` | `powershell -Command "{command}"` |
| Process kill | `SIGTERM` → 2s → `SIGKILL` | `process.Kill()` |
| File permissions | `0644` files, `0755` dirs | OS-determined |
| I2C/SPI tools | Functional (via `/dev/i2c-*`, `/dev/spidev*`) | Returns "Linux only" error |
| Atomic rename | POSIX guarantee | Non-atomic (potential data loss on crash) |

### 4.2 Channel-Specific Edge Cases

| Channel | Edge Case | Handling |
|---|---|---|
| **Discord** | Messages exceeding 2000 chars | Split into multiple messages at natural boundaries |
| **Discord** | Typing indicator | Continuous loop every 8 seconds, max 5 minutes |
| **WeCom Bot** | UTF-8 BOM in messages | Stripped (`\xef\xbb\xbf` prefix removed) |
| **WeCom Bot** | Message deduplication | Buffer of 1000 `msg_id` entries; resets when exceeded |
| **OneBot** | Message deduplication | Ring buffer of 1024 slots; wraps around |
| **OneBot** | WebSocket reconnect | Minimum interval: 5 seconds. Configurable. |
| **OneBot** | Read deadline | 60 seconds. Connection considered dead if no pong. |
| **Telegram** | Placeholder message | Sends "..." first, then edits with actual response |
| **LINE** | Reply token expiry | 25-second window. Falls back to Push API for late responses. |
| **MaixCAM** | Person detection | Converts detection events to inbound messages with bounding box data |

### 4.3 LLM Provider Edge Cases

| Edge Case | Handling |
|---|---|
| Context error from LLM | String-match on `token`, `context`, `invalidparameter`, `length` (case-insensitive) |
| Provider returns both content and tool_calls | Content saved to assistant message alongside tool calls |
| ToolCall with both `function.name` and `name` fields | `NormalizeToolCall()` reconciles provider-specific formats |
| Gemini-specific `thought_signature` and `extra_content` | Copied through to maintain Gemini 3 compatibility |

---

## 5. Technical Debt & Fragile Patterns

### 5.1 Security Debt

| Issue | Severity | Location | Detail |
|---|---|---|---|
| **Shell deny patterns disabled by default** | High | `pkg/config/defaults.go` | `enable_deny_patterns: false` means all shell commands are allowed out of the box. A new user with a misconfigured allow_from could expose unrestricted shell access. |
| **Credentials stored in plaintext** | Medium | `config.json`, `auth/tokens.json` | API keys and OAuth tokens have no encryption at rest. Anyone with filesystem access can read them. |
| **Atomic rename not guaranteed on Windows** | Medium | `pkg/session/manager.go`, `pkg/state/state.go` | `os.Rename()` is not atomic on Windows. A crash during rename could lose session data. |
| **User-Agent string spoofs Chrome** | Low | `pkg/tools/web.go` | `"Mozilla/5.0 ... Chrome/120.0.0.0"` — spoofed UA may violate ToS of some services. |

### 5.2 Hardcoded Magic Numbers

| Value | Location | Risk |
|---|---|---|
| `20` (default max iterations) | `pkg/config/defaults.go` | Configurable, but the interaction between this and context window size is implicit |
| `2.5 chars/token` estimation | `pkg/agent/loop.go` | Inaccurate for non-Latin scripts (CJK can be ~1 char/token). Could trigger premature summarization or miss actual overflows. |
| `3 days` of daily notes | `pkg/agent/context.go` | Not configurable. Users with verbose daily notes may waste context window. Users with sparse notes may want more days. |
| `50%` compression ratio | `pkg/agent/loop.go` | Aggressive. May drop important context. Not configurable. |
| `4 messages` kept after summarization | `pkg/agent/loop.go` | Fixed. Long multi-step workflows lose context. |
| `0.7` Jaccard similarity threshold | `pkg/skills/search_cache.go` | Fixed. May be too aggressive or too lax depending on skill naming conventions. |
| `1000` / `1024` dedup buffer sizes | `pkg/channels/wecom.go`, `pkg/channels/onebot.go` | Fixed. High-traffic channels could overflow. WeCom resets buffer on overflow (lost dedup state). |
| `2000` Discord message limit | `pkg/channels/discord.go` | Matches current Discord API but not externally configured — will need update if Discord changes. |
| `60s` shell timeout | `pkg/tools/shell.go` | Fixed. Long-running builds or analyses will be killed. |
| `10000` chars shell output | `pkg/tools/shell.go` | Fixed. Large build outputs or logs are silently truncated. |
| `50 MB` max ZIP size | `pkg/skills/clawhub_registry.go` | Reasonable but not configurable at the tool level. |
| `120s` summarization timeout | `pkg/agent/loop.go` | Fixed. Complex summaries on slow providers may time out. |

### 5.3 Structural Debt

| Issue | Location | Detail |
|---|---|---|
| **Legacy provider config** | `pkg/config/config.go` | Both `providers` (legacy) and `model_list` (new) sections coexist. Migration logic in `pkg/config/migration.go` maps old format to new, but both paths are maintained. |
| **String-based error detection** | `pkg/agent/loop.go:555` | Context errors identified by substring match (`"token"`, `"context"`, `"invalidparameter"`, `"length"`). Fragile — could false-match or miss new error formats. |
| **Context error message from different providers** | `pkg/agent/loop.go` | Each LLM provider returns context errors in different formats. The substring matching is a brittle compatibility layer. |
| **Zhipu API workaround** | `pkg/agent/loop.go:805` | Compression logic explicitly avoids consecutive system messages "for Zhipu API compatibility". Provider-specific workaround leaks into core logic. |
| **WeCom dedup buffer reset** | `pkg/channels/wecom.go` | When the 1000-entry dedup buffer is full, it's completely cleared. This creates a window where recently-processed messages could be re-processed. |
| **Blocking bus on full buffer** | `pkg/bus/bus.go` | If the 100-message buffer fills, publishers block. In a high-traffic multi-channel scenario, a slow agent could cause all channels to stall. |
| **No graceful degradation for bus backpressure** | `pkg/bus/bus.go` | No overflow policy (drop oldest, drop newest, or alert). Publishers simply block indefinitely. |
| **Session loaded entirely into memory** | `pkg/session/manager.go` | All sessions loaded at startup. For many agents with many peers, this could consume significant memory. No lazy loading or eviction. |
| **Synchronous tool execution** | `pkg/agent/loop.go` | Tools are executed sequentially within a turn, even when independent. Parallel execution would reduce latency for multi-tool turns. |
| **Fixed bootstrap file names** | `pkg/agent/context.go` | `AGENTS.md`, `SOUL.md`, `USER.md`, `IDENTITY.md` are hardcoded. Cannot be customized per agent. |

### 5.4 Missing Features (Implicit from Architecture)

| Gap | Impact |
|---|---|
| **No rate limiting per user** | A single user could monopolize the agent with rapid messages |
| **No message queue persistence** | Bus messages lost on crash (in-memory channels) |
| **No multi-turn tool parallelism** | Each tool call waits for the previous to complete |
| **No per-agent tool restrictions** | All agents share the same tool registry — cannot restrict `exec` for one agent but allow it for another |
| **No audit log** | Tool executions are logged to stdout but not persisted to a queryable format |
| **No webhook replay/retry** | Failed outbound messages are logged but not retried |
| **No session expiry** | Sessions grow indefinitely on disk. No TTL-based cleanup. |
| **No config hot-reload** | Config changes require a gateway restart |

---

## 6. Configuration Defaults Reference

Critical defaults that shape behavior if not explicitly overridden:

| Setting | Default | Risk if Unchanged |
|---|---|---|
| `agents.defaults.model` | `glm-4.7` (Zhipu) | May surprise non-Chinese users expecting GPT/Claude |
| `agents.defaults.restrict_to_workspace` | `true` | Safe default |
| `agents.defaults.max_tool_iterations` | `20` | Reasonable |
| `agents.defaults.max_tokens` | `8192` | May be low for complex tasks |
| `agents.defaults.temperature` | `nil` (provider decides) | Provider-dependent behavior |
| `tools.exec.enable_deny_patterns` | `false` | **Unsafe** — all shell commands allowed |
| `session.dm_scope` | `"main"` | All DM users share one session — potential privacy leak in multi-user setups |
| `heartbeat.enabled` | `true` | Generates periodic agent traffic |
| `heartbeat.interval` | `30` (minutes) | Consumes LLM tokens every 30 minutes |
| `tools.web.duckduckgo.enabled` | `true` | Only search provider enabled by default |
| `tools.skills.registries.clawhub.enabled` | `true` | External registry access enabled |
