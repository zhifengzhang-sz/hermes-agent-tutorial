# Hermes Agent vs. Claude Code — an Honest Comparison for Working Coders

If you already use [Claude Code](https://docs.claude.com/en/docs/claude-code) daily, the natural question is whether Hermes is worth adding to your stack. This is a fair comparison written for working coders — *not* a feature bake-off, because given the same LLM both wrappers produce the same outputs for the same task. The real difference is in *what the wrapper around the LLM lets you do.*

## One-line summary

**Claude Code is a coding agent that lives in your editor. Hermes is an autonomous agent that lives on a server and orchestrates work across channels, schedules, and backends.** If you're at your laptop writing code, Claude Code is usually the better tool. Hermes earns its keep when you're *not* at the laptop — or when you need work to happen on a schedule, on a remote machine, or in response to a message from your phone.

---

## What Hermes genuinely adds that Claude Code doesn't

These are verified features (either against the Hermes source or against a live install):

- **Messaging gateways.** Talk to the agent from Telegram, Discord, Slack, WhatsApp, Signal, Email, SMS, and ~10 more platforms. Useful when you're commuting, in a meeting, away from your keyboard, or on your phone. Claude Code has no equivalent.
- **Scheduled tasks (cron).** Natural-language scheduling: *"every weekday at 8am, summarize yesterday's git commits and send me the result on Telegram."* Stored as JSON at `~/.hermes/cron/jobs.json`, re-read every tick. Claude Code has no built-in scheduler.
- **Multi-provider credential pools with rotation.** Register multiple keys across providers; Hermes automatically fails over on rate limits or billing errors. Claude Code is single-provider (Anthropic).
- **Remote terminal backends.** Hermes can execute its shell commands on a remote SSH host, inside Docker, on Modal serverless, on Daytona sandboxes, or on Singularity HPC clusters — not just locally. Useful for "agent reachable from anywhere, work happens on a cloud VM" setups.
- **Android / Termux support.** First-class `.[termux]` install path. You can run the agent on your phone.
- **Broader bundled skill library.** A fresh install comes with ~75 skills spanning smart home, media, research, MLOps, social, productivity, etc. — not just coding. Claude Code's bundled library is smaller and more coding-centric.
- **Native Home Assistant / smart-home integration.** The agent can control smart-home devices as a native tool.
- **Voice I/O.** Local speech-to-text (faster-whisper) and text-to-speech (Piper or ElevenLabs) ship as optional extras.
- **Image generation tools.** Ship as a built-in toolset.

## Where Claude Code is the better tool

- **Inside your editor.** VS Code, JetBrains, and desktop-app integrations make Claude Code feel native to an IDE-centered workflow. Hermes is CLI/TUI-first and doesn't live in your editor.
- **Coding-specific UX polish.** The multi-file edit flow, patch heuristics, plan mode, permission prompts around risky edits, and session-level git awareness all feel purpose-built for code work. Hermes is a capable generalist that can code, but it isn't tuned to the same fit-and-finish.
- **SDK surface for building on top.** Claude Code has documented Python and TypeScript SDKs (Claude Agent SDK) for embedding agents into other apps. Hermes is more self-contained — less of a platform you build against, more of a finished agent you use.
- **Single-environment simplicity.** Claude Code is one tool, one auth, one install. Hermes has more moving parts (provider, gateway, terminal backend, skills, memory) to configure before it really shines.

## What the marketing overclaims (worth knowing before you buy in)

Hermes's README bills it as a *"self-improving AI agent"* with a *"built-in learning loop."* In practice, the "learning loop" is periodic prompt nudges that ask the LLM to write skill and memory files — see [chapter 4](04-tools-and-skills.md#the-learning-loop--what-it-actually-is) for the full mechanism, or skim the key points:

- **No model weights change.** No fine-tuning, no RLHF, no training of any kind.
- **What happens:** every ~10 turns, Hermes injects a reminder prompt; the LLM may or may not write a file; saved files are re-injected into future sessions' context.
- **Honest description:** context management with a scheduling layer, not learning.

This doesn't make the feature useless — curated skill and memory files are real engineering value — but it isn't what "self-learning" implies. If that's the claim that attracted you to Hermes, evaluate it on what it actually is, not on the marketing framing.

## The complementary pattern (the strongest case for running both)

If you're already a Claude Code user, the most defensible reason to run Hermes is this:

> **Hermes is the orchestrator and interface. Claude Code is the specialist sub-agent.**

Concretely: you chat with Hermes from Telegram, Hermes plans the work, and when the work is coding-heavy it spawns `claude -p "..."` as a subprocess to do the actual implementation. When you invoke `claude` yourself from a shell, calls go against your Claude Code subscription. A subprocess invocation from Hermes *should* present itself to Anthropic's API the same way — same binary, same auth token in `~/.claude/.credentials.json`, same request headers — but this tutorial has **not** verified that empirically, and the related third-party-app billing wall ([chapter 1](01-installation.md#i-already-use-claude-code--do-i-need-a-separate-key)) means the question isn't purely academic. Before building a workflow around this pattern, run one `claude -p "test"` via Hermes and check claude.ai/settings/usage to confirm the spend landed against your subscription rather than Extra usage. Hermes itself is then best run on a cheap non-Anthropic model via OpenRouter or similar, since orchestration work usually doesn't need frontier-tier reasoning.

That pipeline — chat-to-agent from anywhere → plan → delegate heavy coding to Claude Code → notify back on your messaging channel — is real and not replicable with either tool alone, *provided the billing assumption holds on your account*.

## When Hermes isn't worth it

Be honest with yourself about your workflow:

- If your work happens at your desk, in your IDE, from morning to evening, and you have no need for off-laptop access, no recurring scheduled tasks, no multi-provider cost concerns, and no interest in smart-home integration — **Hermes is mostly redundant for you.** Claude Code alone covers your daily work, and the Hermes setup cost (provider, gateway, backend, ~75 bundled skills you won't touch) won't pay back.
- If you were specifically attracted by the "self-learning" claim, read the honest description above and decide whether context management with a nudge scheduler is enough value to you. If not, skip.

## When Hermes earns its keep

- You work from multiple places and want the agent reachable from whichever device is in your hand.
- You have genuinely recurring work that's too irregular for cron but too repetitive to do manually each time.
- You run services on a VPS and want the agent to live there alongside them.
- You want the chat-from-phone → delegate-to-Claude-Code pattern above.
- You care about the specific non-coding integrations (Home Assistant, scheduled media/research, voice, messaging).

## Summary table

| Concern | Claude Code | Hermes |
|---|---|---|
| Writing code at your desk | ✅ Better | ✓ Capable |
| Editor/IDE integration | ✅ Native | ✗ None |
| Coding UX polish (patches, plan mode) | ✅ Tuned for it | ✓ Generalist |
| Off-laptop / phone access | ✗ None | ✅ Multiple gateways |
| Scheduled / recurring tasks | ✗ None | ✅ Built-in cron |
| Multi-provider failover | ✗ None | ✅ Credential pool |
| Remote terminal backends | ✗ Local only | ✅ SSH/Docker/Modal/… |
| Android / Termux | ✗ | ✅ |
| Smart-home / home automation | ✗ | ✅ |
| Voice I/O | ✗ | ✅ (optional extra) |
| "Self-learning" / model improvement | ✗ (no one does this) | ✗ (marketing overclaims context management) |
| SDK for building on top | ✅ Python, TypeScript | ✗ (self-contained) |
| Single-auth simplicity | ✅ | ✗ (more config) |
| Delegation to Claude Code as sub-agent | N/A | ✅ (ch.8) |

---

## Honest bottom line

For a coder who already has Claude Code: Hermes's differentiators are useful but not fundamental. Most of them — messaging, scheduling, remote backends — matter when your work crosses out of "me at my laptop in an IDE" into "me away from the laptop needing work to happen anyway." If that boundary matters to your workflow, Hermes is worth the setup cost. If it doesn't, Claude Code alone is enough.

The "self-learning" claim isn't what makes Hermes useful or not useful. The real value (or lack thereof) is in whether the orchestration layer fits your life.
