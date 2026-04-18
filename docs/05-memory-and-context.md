# 05 — Memory & Context

> **Goal:** understand how Hermes remembers, what it remembers, and how to curate that memory.

---

## What "Memory" Actually Is in Hermes

Three different things get called "memory," and they do different work:

1. **Session history** — the raw turns in the current conversation. Lives in `~/.hermes/sessions/` and in an SQLite database at `~/.hermes/state.db`. Accessible via `hermes -c` and `hermes -r`. The agent can also search *past conversations* via a built-in `session_search` tool backed by an FTS5 full-text index over `state.db`.
2. **Persistent memories** — notes Hermes writes to disk so they survive across sessions. Lives in `~/.hermes/memories/`, in two flat markdown files: `MEMORY.md` (agent-curated facts) and `USER.md` (the profile the agent builds of you). Entries inside each file are separated by a `§` delimiter. Written by the `memory` tool automatically, or manually via `/memory add`.
3. **User model (Honcho)** — if you install the `honcho` extra, Hermes builds a dialectic model of *you* — your preferences, projects, and working style — updated across all sessions.

These stack. Session history is short-term RAM (with full-text search as a retrieval layer on top of old conversations). Persistent memories are long-term notes. Honcho is the psychological profile the agent has of you.

---

## The Auto-Memory Loop

You don't have to teach Hermes to remember — it does it proactively. After tool-rich turns, the agent evaluates whether anything it learned deserves saving. When the answer is yes, it appends a new entry to `~/.hermes/memories/MEMORY.md` (or `USER.md` if the fact is about you), separated from existing entries by `§`.

Unlike the session-search index, memory is **not** retrieved dynamically per-turn. The entire `MEMORY.md` + `USER.md` content is injected into the system prompt as a frozen snapshot at session start, and the model works from that. If you edit the files after a session has already started, the session won't see the change until it's restarted.

Hermes's README frames this as part of a "closed learning loop." The honest description is narrower: **it's prompt-driven file curation, not model learning.** The LLM's weights never change; the agent just periodically writes (and later re-reads) plain markdown files. See [chapter 4](04-tools-and-skills.md#the-learning-loop--what-it-actually-is) for the full mechanism. It's a useful workflow — a curated `MEMORY.md` grows into something like a persistent scratchpad across sessions — but it's context management, not learning.

Sources: `tools/memory_tool.py` for the file format; `run_agent.py:1259` for the memory-nudge interval.

---

## Managing Memories by Hand

In-session commands:

| Command | Does |
|---|---|
| `/memory add "text"` | Force-save a note |
| `/memory search "query"` | Search across all memories (FTS5 in Hermes is used for session history, not memories — memory search is text-level) |
| `/memory list` | List recent memories |
| `/memory delete <id>` | Remove a memory |

Or open `~/.hermes/memories/MEMORY.md` and `USER.md` directly — they're plain markdown with `§`-delimited entries. Edit them with your favorite editor, trim individual entries by hand, or version them with `git`. Changes take effect the next time you start a session.

---

## Session Resumption

Two ways to pick up where you left off:

```bash
hermes -c                          # most recent session
hermes -r "research project"       # fuzzy match by title
```

Session files are stored indefinitely by default. To prune:

```bash
# Keep sessions from the last 30 days, delete older
find ~/.hermes/sessions -name "*.json" -mtime +30 -delete
```

---

## Context Compression

Long sessions eat tokens fast. The `/compress` command is your best friend:

```
/compress
```

Hermes summarizes the middle of the conversation, preserving the first 3 turns (usually context-setting) and the last 4 (usually the active task). The summary is injected in place of the raw turns. (Counts per the [official CLI reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands).)

Good times to compress:

- Before starting a new sub-task in the same session
- When you notice the model "forgetting" earlier context
- Before you walk away — saves tokens on the next resume

---

## Context Files: `SOUL.md` and Project Context

Hermes auto-loads two things from your working directory at session start:

1. **`SOUL.md`** — a personality/system prompt override for this project, applied on top of (or instead of) the global one.
2. **One project context file**, resolved in this priority order — **first match wins, only one is loaded**:
   1. `.hermes.md` or `HERMES.md`
   2. `AGENTS.md`
   3. `CLAUDE.md`
   4. `.cursorrules`

The priority order means Hermes-native files trump tool-agnostic ones, which in turn trump Claude- and Cursor-specific ones. This gives you cross-tool compatibility without having to maintain duplicate files: if you already have a `CLAUDE.md` for another agent, Hermes will pick it up — but a Hermes-specific `.hermes.md` in the same directory will override it.

Note that `MEMORY.md` at the project root is **not** on this list. The `~/.hermes/memories/MEMORY.md` file is the agent's own persistent-memory store (see above), not a project context file. A `MEMORY.md` in your working directory is ignored.

This is how you give the agent project-specific context without polluting the global config. Check into git if the notes are team-shared; `.gitignore` them if they're personal.

Source: `agent/prompt_builder.py` (functions `_load_claude_md`, `build_context_files_prompt`) in the Hermes repo.

---

## Cost & Context Window Realities

Minimum model context: **64K tokens**. Under that, Hermes will refuse to start a session because even the base system prompt plus a couple of tool definitions won't fit.

Rough per-turn budget with the default toolset (numbers from the [NxCode community tutorial](https://www.nxcode.io/resources/news/hermes-agent-tutorial-install-setup-first-agent-2026) — **not** official; treat as ballparks and measure your own):

- **CLI:** 6–8K input tokens per turn
- **Telegram gateway:** 15–20K per message (extra framing + gateway protocol)
- **With heavy skills loaded:** add 1–3K per active skill

If cost matters, you have four levers:

1. Use a cheaper model (OpenRouter has models 10–100× cheaper than frontier)
2. Disable tool categories you don't need: `hermes tools`
3. `/compress` aggressively
4. Keep `SOUL.md` lean — it's loaded on every turn

---

## Try This

Run this experiment to see the memory loop in action:

1. Start a session: `hermes`
2. Say: "I prefer pnpm over npm for all JavaScript projects. Remember this."
3. Quit: `/quit`
4. Start a fresh session: `hermes`
5. Say: "Set up a new React project for me."
6. Watch Hermes use `pnpm create vite@latest` rather than `npm create`.

If Hermes doesn't remember, check `~/.hermes/memories/MEMORY.md` — the preference should appear as a new `§`-delimited entry. If it isn't there, the model may have decided the fact wasn't worth persisting; be more explicit: `/memory add "I prefer pnpm over npm for all JS projects"`.

---

**Next →** [06 — Messaging Gateways](06-messaging-gateways.md)
