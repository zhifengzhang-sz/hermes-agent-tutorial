# REVIEW.md — Verification Ledger

This tutorial makes factual claims about Hermes Agent's internals, configuration, and behavior. Not all of them are equally grounded. This file is the honest ledger of which is which, so readers can tell apart "checked against the source" from "sourced from a third-party writeup" from "educated guess."

Updated: **2026-04-17**.

---

## ✅ Verified against the Hermes source tree

These claims have been cross-checked against [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent). Source file references are cited inline in the relevant chapter.

| Claim | Chapter | Source |
|---|---|---|
| `claude-code` skill ships at `skills/autonomous-ai-agents/claude-code/` in-repo and installs to `~/.hermes/skills/autonomous-ai-agents/claude-code/` (category subpath preserved) | [08](docs/08-claude-code-integration.md) | `tools/skills_sync.py` |
| Context files auto-loaded from CWD at session start, priority order: `.hermes.md`/`HERMES.md` → `AGENTS.md` → `CLAUDE.md` → `.cursorrules`, first match wins, only one is loaded | [05](docs/05-memory-and-context.md) | `agent/prompt_builder.py` |
| `MEMORY.md` at project root is **not** auto-loaded; `~/.hermes/memories/MEMORY.md` is the agent's own persistent-memory store | [05](docs/05-memory-and-context.md) | `tools/memory_tool.py` |
| Persistent memory is stored in two flat markdown files (`MEMORY.md`, `USER.md`) with `§`-delimited entries, injected into the system prompt as a frozen snapshot at session start | [05](docs/05-memory-and-context.md) | `tools/memory_tool.py` |
| FTS5 index exists, but it's over *session messages* in `~/.hermes/state.db` used by `session_search` — **not** over memory entries | [05](docs/05-memory-and-context.md) | `hermes_state.py` |
| Cron jobs live in a single JSON file at `~/.hermes/cron/jobs.json`, not per-task YAMLs | [07](docs/07-scheduled-tasks.md) | `cron/jobs.py` |
| Cron job `deliver` field is a flat string (`"telegram"`, `"local"`, etc.), not a nested object | [07](docs/07-scheduled-tasks.md), [telegram-morning-briefing](examples/telegram-morning-briefing/README.md) | `cron/jobs.py` |
| Per-cron `model` and `provider` overrides are top-level scalar fields, not a nested `provider:` object | [07](docs/07-scheduled-tasks.md), [telegram-morning-briefing](examples/telegram-morning-briefing/README.md) | `cron/jobs.py`, `tools/cronjob_tools.py` |
| Installer extras table (`messaging`, `cron`, `voice`, `tts-premium`, `mcp`, `honcho`, `pty`, `modal`, `dev`, `termux`) — all listed extras exist | [01](docs/01-installation.md) | `pyproject.toml` |
| Skill discovery at runtime scans only `~/.hermes/skills/` plus `skills.external_dirs` from config — the bundled repo `skills/` directory is **not** scanned at runtime, only used as a seed source for install-time sync | [04](docs/04-tools-and-skills.md) | `tools/skills_tool.py`, `agent/skill_utils.py` |
| Hermes auto-imports Claude Code's login from `~/.claude/.credentials.json` into its credential pool as an Anthropic OAuth entry (`source: claude_code`). `hermes status` does **not** surface pool-based OAuth credentials — `hermes auth list` is the authoritative view | [01](docs/01-installation.md) | `website/docs/user-guide/features/credential-pools.md`; confirmed via `hermes auth list` on a real install |
| Anthropic's current (2026-04) billing policy routes third-party apps like Hermes against the user's "Extra usage" balance at claude.ai/settings/usage, **not** against Max/Pro subscription plan limits. An imported Claude Code OAuth credential will fail with HTTP 400 at first call if Extra usage is unfunded — the error body reads "Third-party apps now draw from your extra usage, not your plan limits. Add more at claude.ai/settings/usage and keep going." | [01](docs/01-installation.md) | Verified live via HTTP 400 from `api.anthropic.com` on 2026-04-18 |
| Installer default checkout path is `~/.hermes/hermes-agent` (code lives inside the data dir); overridable via `--dir` / `HERMES_INSTALL_DIR` | [09](docs/09-troubleshooting.md) | `scripts/install.sh:32,82` |
| Cron scheduler re-reads `jobs.json` on every tick — hand edits are picked up without a gateway restart | [07](docs/07-scheduled-tasks.md), [telegram-morning-briefing](examples/telegram-morning-briefing/README.md) | `cron/jobs.py:320,664-673` (`get_due_jobs()` calls `load_jobs()`) |
| Missed-while-down recurring jobs are fast-forwarded to their next future run (no backlog replay) rather than firing a burst on restart | [07](docs/07-scheduled-tasks.md) | `cron/jobs.py:664-670` docstring + logic |
| The bundled `claude-code` skill orchestrates interactive Claude Code via tmux with specific pane sizing, first-launch dialog handling (workspace trust + optional permissions warning), and capture-pane polling. Tutorial prose summarizes the shape; the skill at `skills/autonomous-ai-agents/claude-code/SKILL.md` (v2.2.0) is authoritative | [08](docs/08-claude-code-integration.md), [claude-code-delegation](examples/claude-code-delegation/README.md) | `skills/autonomous-ai-agents/claude-code/SKILL.md` |
| The Hermes "learning loop" is **not** model-weight learning. It is: (a) a turn-counter (`_memory_nudge_interval` / `_skill_nudge_interval`, default 10 each, configurable) that periodically injects a reminder prompt, plus (b) a `skill_manager` tool that the LLM may choose to call to write a `SKILL.md` file. Saved files are re-loaded into future sessions' context (skills on-demand, memory as a frozen snapshot). No training, no fine-tuning, no weight updates. The README's "self-improving agent" and "built-in learning loop" framing overstates this; the tutorial (ch.4) now describes the actual mechanism plainly | [04](docs/04-tools-and-skills.md), [hermes-vs-claude-code](docs/hermes-vs-claude-code.md) | `run_agent.py:1259,1383-1387,7867`; `tools/skill_manager_tool.py`; verified by in-session grep on 2026-04-18 |

## ⚠️ Attributed but not verified

These claims come from third-party sources. They're flagged inline where they appear.

| Claim | Chapter | Source |
|---|---|---|
| `/compress` preserves the first 3 and last 4 turns | [05](docs/05-memory-and-context.md) | Official Hermes CLI reference — cited inline |
| 6–8K CLI tokens / 15–20K Telegram tokens per turn | [03](docs/03-cli-essentials.md), [05](docs/05-memory-and-context.md) | [NxCode community tutorial](https://www.nxcode.io/resources/news/hermes-agent-tutorial-install-setup-first-agent-2026) |

## ❓ Still needs a clean-VM pass

These claims are behavior-level and couldn't be validated from reading source alone. They need to be run on a fresh install to confirm (or correct).

- Natural-language cron parsing producing the exact cron expressions in the [ch.7 table](docs/07-scheduled-tasks.md#schedule-syntax)
- The `hermes -s skill1,skill2` per-session skill-load flag behavior
- The `/background <task>` parallel session mechanic
- Gateway auto-approve semantics and what happens when a destructive tool call fires from Telegram
- The Claude Code interactive-mode tmux orchestration trace in [ch.8](docs/08-claude-code-integration.md) — specifically, the exact `send-keys`/`capture-pane` loop the bundled skill uses
- **Whether a `claude` subprocess spawned by Hermes bills against the user's subscription (as a direct `claude` invocation would) or against the third-party-app "Extra usage" bucket.** Central to the complementary pattern in [hermes-vs-claude-code.md](docs/hermes-vs-claude-code.md); currently framed as an expectation, not a verified fact. Trivial to check on a real install: run one `claude -p "test"` via Hermes, then inspect claude.ai/settings/usage.
- Installer idempotency and the Termux fallback path
- Image paste (`Ctrl+V`) and voice (`Ctrl+B`) key bindings on Linux, macOS, and Termux

If you run any of these end-to-end, open a PR striking the entry from this list and citing the command transcript.

## Known omissions

These are things a tutorial of this scope should eventually cover but currently doesn't:

- Ollama / llama.cpp / vLLM local-model specifics — only mentioned in passing
- Per-task dollar-cost estimation (we have token budgets, not costs)
- Honcho setup walkthrough — referenced in ch.5 but not demonstrated
- Discord / Slack / WhatsApp / Signal full gateway setups — only Telegram has a worked example

---

## How to help

Found a claim that's wrong, outdated, or drifting from the official docs? Open a PR against the relevant chapter and — if the claim is structural — cite the Hermes source file. If it's behavioral, paste a terminal transcript.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the broader contribution rules.
