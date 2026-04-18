# 04 — Tools & Skills

> **Goal:** understand the difference, and write your first custom skill.
>
> **Official sources:** [Skills Hub](https://hermes-agent.nousresearch.com/docs/skills/) · [agentskills.io](https://agentskills.io)

---

## Tools vs. Skills (the one-minute version)

**Tools** are single functions the model can call — `read_file`, `run_command`, `web_search`. They're atomic, written in Python, and live inside the Hermes codebase.

**Skills** are bundles of instructions, prompts, and optional sub-tools packaged as a folder. They're markdown-first, portable, and shareable via [agentskills.io](https://agentskills.io). Think of a skill as a **playbook** that tells the agent *how* to use existing tools to accomplish a specific kind of task.

Rule of thumb: if you're adding a new capability (e.g., "talk to Asana's API"), write a tool. If you're teaching the agent a workflow (e.g., "how to review a pull request at our company"), write a skill.

---

## Browsing the Skills Hub

```bash
# Inside a Hermes session
/skill search github
/skill install github-pr-workflow
```

Or browse visually at [hermes-agent.nousresearch.com/docs/skills](https://hermes-agent.nousresearch.com/docs/skills/). Notable skills shipped today:

- **`claude-code`** — delegates coding tasks to Anthropic's Claude Code CLI (covered in chapter 8)
- **`codex-cli`** — same thing for OpenAI Codex
- **`github-auth`** — sets up GitHub credentials (HTTPS tokens, SSH keys, `gh` CLI)
- **`github-pr-review`** — analyzes diffs and leaves inline PR comments
- **`github-issues`** — creates, triages, and closes issues
- **`pygount`** — codebase stats (LOC, language breakdown)

Each skill is a folder under `~/.hermes/skills/` containing at minimum a `SKILL.md`. User-created skills typically live directly at `~/.hermes/skills/<skill-name>/`, while bundled skills preserve their category subpath from the Hermes repo (e.g., `~/.hermes/skills/autonomous-ai-agents/claude-code/SKILL.md`).

---

## The "Learning Loop" — What It Actually Is

Hermes Agent's [upstream README](https://github.com/NousResearch/hermes-agent) calls this a **"built-in learning loop"** and pitches the agent as *"self-improving."* Both phrases overstate what the system does, and it's worth being direct about it before you evaluate the feature. (This tutorial's own README doesn't use those phrases — this callout is correcting upstream's framing, not the tutorial's.)

**What does not happen.** Model weights never change. There is no fine-tuning, no RLHF, no gradient update, no continual pre-training. The LLM you point Hermes at is bit-for-bit the same LLM after any number of sessions. Whatever "learning" means in ML, that isn't happening here.

**What actually happens,** verified in `run_agent.py:1259,1383-1387` and `tools/skill_manager_tool.py` in the Hermes source:

1. Hermes maintains two turn counters: `_memory_nudge_interval` (default 10) and `_skill_nudge_interval` (default 10). Both configurable in `~/.hermes/config.yaml` under `memory.nudge_interval` and `skills.creation_nudge_interval`.
2. Every N turns, Hermes injects a short reminder prompt into the agent's context — asking the model to consider whether a recent pattern is worth saving as a skill, or whether a fact is worth writing to memory.
3. The LLM reads that reminder like any other text. It may respond by calling the `skill_manager` tool to write a `SKILL.md` to `~/.hermes/skills/`, or by appending an entry to `~/.hermes/memories/MEMORY.md`. Or it may do nothing.
4. On future sessions, those saved files are loaded back into the agent's context (skills on demand via progressive disclosure; memory as a frozen snapshot at session start).
5. Counters reset when the corresponding tool is actually used (`run_agent.py:7867`).

**In plain engineering terms:** this is **prompt injection + file writes + context retrieval.** It's a form of *context management* — automating the "write yourself a note you'll see later" pattern. Nothing more. Calling it "learning" stretches the word past the point where it usefully applies.

**Is the pattern useful?** It can be. Having the agent periodically prompted to consider persisting knowledge is a reasonable workflow, and a curated `~/.hermes/skills/` directory can grow into a genuinely useful personal library over time. But the value depends entirely on the LLM's judgment in the moment a nudge fires: a weak model saves junk skills or misses good ones; a strong model does better. The mechanism itself adds no model capability — it adds scheduling around the tool-use the model can already do.

**You can drive it manually too,** which is arguably the more reliable path:

```
/skill create
```

This walks you through capturing the current session's approach into a reusable skill file. Same underlying mechanism (model writes a SKILL.md via the `skill_manager` tool), just triggered explicitly instead of by a timed nudge.

See also the [Hermes vs. Claude Code](hermes-vs-claude-code.md) comparison for the broader context on how to weigh this feature against Hermes's other capabilities.

---

## Writing Your First Skill

Skills are just markdown. The minimum viable skill is:

```markdown
---
name: my-skill
description: One-sentence description of when to use this skill.
---

# My Skill

Instructions to the agent written in second person.

## When to use
Bullet list of triggers.

## Steps
1. First action
2. Second action
3. ...
```

Drop that at `~/.hermes/skills/my-skill/SKILL.md` and restart Hermes. It's now available via `/my-skill` and, if the description matches a user prompt, will be invoked automatically.

See [`examples/hello-skill/`](../examples/hello-skill/) in this repo for a working minimal skill you can copy.

---

## Skill Design Tips

A few patterns that survive real use:

1. **Make the description specific, not generic.** "When to use: any GitHub question" is noise. "When to use: reviewing diffs on a PR that touches TypeScript files" is signal.
2. **Write steps as imperatives.** "Run `pytest -k test_auth`" beats "you might want to consider running tests."
3. **Include negative examples.** "Do NOT use `pip install` — this project uses `uv`." The agent respects clear prohibitions.
4. **Reference tools explicitly.** If step 2 should use the `read_file` tool, say so.
5. **Keep it short.** Skills go into the system prompt at activation time; every token costs money on every turn the skill is active.

---

## MCP Servers

In addition to skills, Hermes speaks the **Model Context Protocol**. If you install the `mcp` extra:

```bash
uv pip install -e ".[mcp]"
```

…you can plug in any MCP server (Asana, Gmail, Salesforce, your own) and its tools become available to the agent. Configure in `~/.hermes/config.yaml`:

```yaml
mcp:
  servers:
    - name: asana
      url: https://mcp.asana.com/sse
    - name: my-local-server
      command: python
      args: [-m, my_mcp_server]
```

This is the escape hatch for "I need Hermes to talk to system X" when no skill exists yet.

---

## Try This

Build a skill that captures *your* workflow. Good first-skill candidates:

- **`standup-notes`** — reads your git log for the last 24 hours and drafts your daily standup
- **`pr-summary`** — takes a PR URL and produces a Slack-ready summary
- **`project-bootstrap`** — creates a new project directory with your preferred scaffolding

Each one should fit in under 30 lines of markdown. If yours is getting longer, it probably wants to be split into multiple skills.

---

**Next →** [05 — Memory & Context](05-memory-and-context.md)
