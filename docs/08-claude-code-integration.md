# 08 — Claude Code Integration

> **Goal:** have Hermes delegate heavy coding work to Anthropic's [Claude Code](https://docs.claude.com/en/docs/claude-code/overview), orchestrating it as a sub-agent.
>
> **Official source:** [`skills/autonomous-ai-agents/claude-code/SKILL.md`](https://github.com/NousResearch/hermes-agent/blob/main/skills/autonomous-ai-agents/claude-code/SKILL.md) in the Hermes repo.

---

## Why Delegate to Claude Code?

Hermes is a great generalist, but for long, multi-step coding tasks — refactors, migrations, PR reviews — a **purpose-built coding agent** tends to outperform a generalist with file-edit tools. Claude Code was built for exactly this: it has tighter editing heuristics, smarter sub-agent spawning, and git-aware workflows.

The combination:

```
You ──▶ Hermes (generalist, messaging, memory)
           │
           └──▶ Claude Code (specialist, coding)
                     │
                     └──▶ your repo
```

You can talk to Hermes from Telegram, have it plan the high-level task, delegate the implementation to Claude Code, then report results back.

---

## Prerequisites

1. **Claude Code installed.** See the [official install guide](https://docs.claude.com/en/docs/claude-code/quickstart).
   ```bash
   npm install -g @anthropic-ai/claude-code
   claude --version
   ```
2. **Claude Code authenticated.** Run `claude` once interactively to log in. The credentials are cached in `~/.claude/`.
3. **Hermes installed with the `pty` extra** — interactive Claude Code sessions need a PTY.
   ```bash
   uv pip install -e ".[pty]"
   ```
4. **`tmux` installed** (optional but recommended for interactive mode):
   ```bash
   # macOS
   brew install tmux
   # Debian/Ubuntu/WSL2
   sudo apt install tmux
   ```

The `claude-code` skill ships with Hermes — you don't need to install it separately. It's bundled at `skills/autonomous-ai-agents/claude-code/SKILL.md` in the repo, and the installer copies it (preserving the category subpath) into your user skills directory on first run. Check:

```bash
ls ~/.hermes/skills/autonomous-ai-agents/claude-code/
```

If that path is empty, the bundled-skills sync didn't run — re-run the installer, or run `hermes doctor` to diagnose.

---

## Two Modes

### Print Mode (one-shot, no PTY)

This is the cleanest integration. Hermes shells out to `claude -p`, which runs a single task and exits. No interactive prompts, no permission dialogs.

Typical call Hermes makes on your behalf:

```bash
claude -p "Add error handling to all API calls in src/" \
  --allowedTools "Read,Edit" \
  --max-turns 10
```

**Use print mode for:** refactors with a clear spec, test generation, adding type hints, fixing a specific bug described by URL or issue number.

### Interactive Mode (full TUI, requires tmux)

For longer tasks where you want Claude Code to plan, spawn sub-agents, and take more than 10 turns. Hermes opens Claude Code in a `tmux` session, watches output via `capture-pane`, and sends input via `send-keys`.

**Use interactive mode for:** exploratory work, multi-file features, anything where the task definition might shift mid-run.

---

## Your First Delegation

Open a Hermes session:

```bash
hermes
```

Then ask it to delegate a task. Be explicit about using Claude Code:

```
> Use Claude Code to add input validation to all POST endpoints in
  ~/code/my-api/src/routes. Allow only Read and Edit tools. Limit to 15 turns.
```

Hermes will:

1. Load the `claude-code` skill into context (the skill tells it the command shape)
2. Issue a `claude -p ...` command via its terminal tool
3. Stream Claude Code's output back into the Hermes session
4. Summarize the diff and any failures

If you're on Telegram, you get the summary there. If you're on CLI, you see it streamed.

---

## Important Flags & Gotchas

From the `claude-code` skill's own field notes, these are the ones that bite:

| Flag / behavior | What to know |
|---|---|
| `--dangerously-skip-permissions` | Dialog defaults to **"No, exit"**. In interactive mode Hermes sends `Down` then `Enter` to accept. Print mode skips it entirely. |
| `--max-budget-usd` | Minimum is ~$0.05. System prompt cache creation alone costs this. Setting lower errors immediately. |
| `--max-turns` | **Print mode only.** Ignored in interactive sessions. |
| `--json-schema` | Needs enough `--max-turns` — Claude must read files before producing structured output. |
| Session resumption | `--continue` finds the most recent session *for the current working directory*. Run it from the same `cwd` you started in. |
| `python` vs `python3` | On systems without a `python` symlink, Claude's bash commands may fail on the first try. It self-corrects, but expect one retry. |

---

## A Concrete Recipe

Say you want Hermes to review an open PR on GitHub, apply the suggested changes, and push. Here's the orchestration:

```
> I have a PR at https://github.com/me/myproject/pull/42 with review comments.
  Clone it fresh into /tmp/pr-42-review, use Claude Code to apply the
  requested changes, run tests, commit, and push. Use interactive mode
  because there are multiple files.
```

Hermes translates this into roughly the following shape. These are the conceptual steps, not the exact commands — the authoritative sequence (tmux pane sizing, first-launch dialog handling, capture-pane polling, timing) lives in the bundled skill at `~/.hermes/skills/autonomous-ai-agents/claude-code/SKILL.md` (v2.2.0 at the time of writing). Read that for the commands Hermes actually issues.

1. Check out the PR locally (`gh pr checkout 42`)
2. Fetch the PR's review comments from GitHub at runtime (`gh pr view 42 --json comments,reviews`) — PR comments live on the PR via the API, not in any file in the repo
3. Start an interactive Claude Code session inside a tmux pane
4. Handle Claude Code's first-launch dialogs — workspace trust, and (if you passed `--dangerously-skip-permissions`) the permissions warning
5. Send the instruction plus the fetched feedback as a single prompt
6. Poll `tmux capture-pane` until Claude Code reports idle or repeats an error
7. Run tests, commit, push
8. Summarize back to you

(Earlier drafts of this guide invented a `.github/pr-review.md` file as the comment source — that's not how GitHub works. Fetch at runtime via `gh` as above.)

See [`examples/claude-code-delegation/`](../examples/claude-code-delegation/) for the full worked example with prompts and expected output.

---

## Orchestration Prompts That Work

A few templates:

**Bounded print-mode task:**
> "Use Claude Code in print mode. Task: `<one-sentence spec>`. Tools: `Read,Edit`. Max turns: 10. If it can't finish, summarize why."

**Iterative refactor:**
> "Delegate to Claude Code interactively. Refactor `<file>` to use `<pattern>`. After each edit, have it run `<test command>`. If tests fail, have it fix and re-run. Stop when green or after 5 attempts."

**Review-driven work:**
> "Have Claude Code review `<PR URL>`, list issues as a markdown checklist, then work through the list one by one. Commit after each item."

---

## When NOT to Delegate

Orchestration overhead is real. Don't delegate to Claude Code when:

- The task is a single file edit — Hermes' own file tools are faster
- You need full control over every tool call — Claude Code runs semi-autonomously
- Budget is tight — you're paying for two models (Hermes' planner + Claude Code's actor)
- The project is tiny — setup cost exceeds the work

Heuristic: delegate when the task would take Hermes 20+ tool calls on its own.

---

**Next →** [09 — Troubleshooting](09-troubleshooting.md)
