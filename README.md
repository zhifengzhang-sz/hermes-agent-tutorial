# Hermes Agent — A Hands-On Tutorial

> A guided tour of [Hermes Agent](https://hermes-agent.nousresearch.com) by Nous Research — from install, to your first conversation, to delegating coding tasks to Claude Code, to running a Telegram-driven autonomous assistant that never forgets.

This is a **community tutorial**, not official documentation. Every chapter links back to the authoritative source on [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/docs/) or the [GitHub repo](https://github.com/NousResearch/hermes-agent). If something here conflicts with the official docs, the docs win.

> **Verification status (2026-04-18):** *Structural claims* (file paths, config schemas, auto-loaded files, skill discovery, credential-pool behavior, cron scheduler internals, memory file format) are cross-checked against the [Hermes source tree](https://github.com/NousResearch/hermes-agent) and cited inline. *Runtime behaviors live-verified on a working install* include: the Ollama provider path plus Hermes's 64K-context-floor rejection, the OpenRouter API-key path and the free-tier rate-limit reality, the Claude Code OAuth auto-import and its Extra-usage HTTP 400 wall, the `claude -p` subprocess delegation, and Anthropic's `apiProvider: "firstParty"` first-party billing designation. *Still unverified against a running install — treat as ballparks:* per-turn token-budget numbers (community-sourced from NxCode), `/compress` exact turn-preservation behavior, and empirical skill-nudge cadence under real workloads (the 10-turn default is source-verified but not load-tested). The full ledger of what's verified vs. live-tested vs. pending lives in [`REVIEW.md`](REVIEW.md).

---

## Why Hermes?

Hermes Agent is an **autonomous, self-hosted AI agent** that:

- Lives on your machine (laptop, VPS, or Android phone via Termux)
- Works through any LLM you want — OpenRouter, Nous Portal, OpenAI, or a local endpoint
- Reaches you on Telegram, Discord, Slack, WhatsApp, Signal, or email
- Curates a personal skill + memory library as you use it (prompt-driven file writes — *not* model fine-tuning; see the [honest comparison](docs/hermes-vs-claude-code.md) and [chapter 4](docs/04-tools-and-skills.md#the-learning-loop--what-it-actually-is) for what this does and doesn't do)
- Keeps a persistent memory across sessions
- Can spawn sub-agents like **Claude Code** for heavy coding tasks

If you've used OpenClaw, think of Hermes as the OpenClaw successor with a similar skill-curation mechanism and a smaller attack surface.

**Already a Claude Code user?** Read [Hermes vs. Claude Code](docs/hermes-vs-claude-code.md) first. Short version: Hermes is useful-but-not-fundamental for coders who already have Claude Code; it earns its keep when you need off-laptop access, scheduling, or remote backends, and it pairs well with Claude Code via the [delegation pattern](docs/08-claude-code-integration.md).

---

## What You'll Learn

By the end of this tutorial you will be able to:

1. Install Hermes on Linux, macOS, WSL2, or Termux
2. Configure an LLM provider and enable the tools you actually need
3. Navigate the TUI — slash commands, multiline input, session resumption
4. Understand tools vs. skills, and write your first custom skill
5. Hook Hermes to Telegram so it reaches you anywhere
6. Delegate coding work to Claude Code as a sub-agent
7. Schedule natural-language cron jobs ("every morning at 9, summarize HN")
8. Diagnose and fix the most common problems

---

## Prerequisites

- A Unix-like shell (Linux, macOS, WSL2, or Termux on Android)
- `git` installed (`git --version` should work)
- A credential for **one** LLM provider — not from Hermes itself, but from the model provider you'll route through (OpenRouter, Anthropic, OpenAI, Nous Portal, local Ollama, etc.). [OpenRouter](https://openrouter.ai) is the easiest starting point; [chapter 1](docs/01-installation.md#before-you-start) walks through the options, including what to do if you already use Claude Code.
- About 30 minutes for the core path, 2 hours for everything

You do **not** need to install Python, Node.js, `ripgrep`, or `ffmpeg` yourself — the Hermes installer handles those.

---

## How to Use This Tutorial

Two paths:

**Linear path (recommended for first-timers)** — read chapters 1 → 9 in order. Each one builds on the last and takes 10–20 minutes.

**Goal path** — jump to what you need:

| I want to... | Start here |
|---|---|
| Just get it running | [01 — Installation](docs/01-installation.md) |
| Pick an LLM provider (Ollama / OpenRouter / Claude Code delegation) | [LLM Provider Scenarios](docs/llm-provider-scenarios.md) |
| Understand the CLI | [03 — CLI Essentials](docs/03-cli-essentials.md) |
| Write my own skill | [04 — Tools & Skills](docs/04-tools-and-skills.md) |
| Use Hermes from my phone | [06 — Messaging Gateways](docs/06-messaging-gateways.md) |
| Delegate to Claude Code | [08 — Claude Code Integration](docs/08-claude-code-integration.md) |
| Run scheduled tasks | [07 — Scheduled Tasks](docs/07-scheduled-tasks.md) |
| Fix something that broke | [09 — Troubleshooting](docs/09-troubleshooting.md) |

---

## Table of Contents

1. [Installation](docs/01-installation.md)
2. [Your First Conversation](docs/02-first-conversation.md)
3. [CLI Essentials](docs/03-cli-essentials.md)
4. [Tools & Skills](docs/04-tools-and-skills.md)
5. [Memory & Context](docs/05-memory-and-context.md)
6. [Messaging Gateways](docs/06-messaging-gateways.md)
7. [Scheduled Tasks](docs/07-scheduled-tasks.md)
8. [Claude Code Integration](docs/08-claude-code-integration.md)
9. [Troubleshooting](docs/09-troubleshooting.md)

### Reference

- [Hermes vs. Claude Code — Honest Comparison](docs/hermes-vs-claude-code.md) — if you already use Claude Code, read this first
- [LLM Provider Scenarios](docs/llm-provider-scenarios.md) — three end-to-end setups (Ollama / OpenRouter / OpenRouter + Claude Code delegation), each tested on a real install with findings documented

### Examples

- [`examples/hello-skill/`](examples/hello-skill/) — a minimal custom skill you can drop into `~/.hermes/skills/`
- [`examples/claude-code-delegation/`](examples/claude-code-delegation/) — a recipe for delegating a refactor to Claude Code
- [`examples/telegram-morning-briefing/`](examples/telegram-morning-briefing/) — a scheduled Telegram task

### Exercises

- [`exercises/README.md`](exercises/README.md) — progressively harder challenges, from "get it running" to "ship a production cron job"

---

## Sources & Credit

This tutorial is built on material from:

- **Official docs** — [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/) (MIT-licensed)
- **GitHub repo** — [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- **Skills Hub** — [agentskills.io](https://agentskills.io)
- **DataCamp tutorial** by community contributors — [datacamp.com/tutorial/hermes-agent](https://www.datacamp.com/tutorial/hermes-agent)
- **NxCode walkthrough** — [nxcode.io](https://www.nxcode.io/resources/news/hermes-agent-tutorial-install-setup-first-agent-2026)
- **DeepWiki** (auto-indexed source) — [deepwiki.com/NousResearch/hermes-agent](https://deepwiki.com/NousResearch/hermes-agent)

All prose in this tutorial is original. Command snippets are reproduced from the docs because commands are facts, not prose.

---

## Contributing

Found a bug, typo, or outdated step? PRs welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
