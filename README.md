# santaclaws

A read-only knowledge base containing detailed technical blueprints of five [OpenClaw](https://github.com/openclaw/openclaw) ecosystem implementations. This repository is designed to be explored by AI coding assistants (Claude Code, Cursor, GitHub Copilot) to answer comparative questions about these projects.

> **This repository contains no runnable code.** It is a collection of architectural documentation meant for AI-assisted research and comparison.

## Implementations

| Project | Language | Description |
|---------|----------|-------------|
| **[openclaw/](openclaw/)** | TypeScript/Node.js | The original — a self-hosted, multi-channel AI gateway bridging LLMs with messaging platforms (Telegram, Discord, Slack, WhatsApp, Signal, etc.). Created by Peter Steinberger. |
| **[zeroclaw/](zeroclaw/)** | Rust | Zero-overhead Rust reimplementation with swappable trait-based subsystems, hybrid vector+FTS memory, and optional PostgreSQL. ~3.4 MB binary, <10 ms startup. |
| **[nanoclaw/](nanoclaw/)** | TypeScript/Node.js | Minimal single-user assistant — WhatsApp-only, no HTTP server, runs AI agents inside Docker/Apple containers with file-based IPC. |
| **[ironclaw/](ironclaw/)** | Rust | Security-focused agent runtime with WASM-sandboxed tool execution, prompt injection defense, PostgreSQL + libSQL storage, and a terminal UI (Ratatui). |
| **[picoclaw/](picoclaw/)** | Go | Ultra-lightweight — single static binary, <10 MB RAM, <1 s startup, runs on $10 hardware. Supports 12+ messaging platform SDKs. |

## Blueprint Structure

Each implementation directory contains 5 markdown files:

| File | Contents |
|------|----------|
| `01_System_Overview.md` | Project identity, tech stack, architecture, deployment |
| `02_Data_Model.md` | Database schemas, data structures, storage patterns |
| `03_Core_Workflows.md` | Message flow, agent execution, key runtime paths |
| `04_API_and_Integrations.md` | APIs, webhooks, platform connectors, protocols |
| `05_Domain_Logic.md` | Business rules, scheduling, memory, domain-specific logic |

## How to Use This Repository

Clone the repo and open it in an AI coding assistant:

```bash
git clone https://github.com/harikb/santaclaws.git
cd santaclaws

# Claude Code
claude

# Cursor
cursor .
```

Then ask questions like:

- "What unique features does IronClaw have that ZeroClaw doesn't?"
- "Which implementation would you recommend for a small community of 50 members?"
- "Compare the security models across all five implementations"
- "Which projects support WhatsApp, and how do their approaches differ?"
- "What databases does each implementation use? Summarize in a table"
- "I want to run an AI assistant on a Raspberry Pi — which one should I pick?"
- "How does each project handle agent memory and context persistence?"

## Important

- **Do not edit the blueprint files.** They are reference documentation, not source code.
- **This repo has no build steps, tests, or dependencies.** It is pure documentation.
- If you want to contribute a new implementation blueprint, follow the existing 5-file structure.
