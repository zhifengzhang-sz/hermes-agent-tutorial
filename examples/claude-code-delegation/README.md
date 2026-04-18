# Example: Delegating a PR Review to Claude Code

A worked example of using Hermes to orchestrate Claude Code for a real coding task — reviewing an open PR, applying the reviewer's feedback, and pushing the changes back.

This is the concrete version of the recipe in [`docs/08-claude-code-integration.md`](../../docs/08-claude-code-integration.md).

---

## Scenario

You have a PR at `https://github.com/me/myproject/pull/42`. The reviewer left 5 comments asking for:

1. Input validation on two endpoints
2. Replacing `console.log` with the project's logger
3. Adding tests for the new function
4. A docstring on the exported API
5. Renaming a variable for clarity

You're on Telegram, away from your laptop. You want this done before your next meeting.

---

## Prerequisites

- Hermes installed with `pty` and `messaging` extras
- Claude Code installed and authenticated (`claude --version` works)
- Telegram gateway set up and running
- `tmux` installed

---

## The Prompt

From Telegram, send:

```
Clone https://github.com/me/myproject/pull/42 fresh into /tmp/pr-42.
Use Claude Code in interactive mode to apply the PR review comments.
After each fix, run `pnpm test`. When all comments are addressed and
tests pass, commit with message "Apply PR review feedback" and push.
If tests fail twice in a row on the same fix, stop and summarize.
```

---

## What Hermes Does Under the Hood

An *illustrative* trace of tool calls — these are representative commands, not the exact sequence Hermes issues. The bundled `claude-code` skill (`~/.hermes/skills/autonomous-ai-agents/claude-code/SKILL.md`) owns the real orchestration details: tmux pane sizing, first-launch dialog handling, `capture-pane` scroll-back, inter-step timing. Read the skill file for the authoritative pattern.

```
 1. terminal: gh repo clone me/myproject /tmp/pr-42
 2. terminal: cd /tmp/pr-42 && gh pr checkout 42
 3. terminal: gh pr view 42 --json comments,reviews --jq \
      '.comments[].body, .reviews[].comments[].body' > /tmp/pr-42-feedback.txt
 4. terminal: cd /tmp/pr-42 && tmux new-session -d -s claude-pr42 -x 140 -y 40
 5. terminal: tmux send-keys -t claude-pr42 'cd /tmp/pr-42 && claude' Enter
 6. (wait ~3–5s for Claude Code's welcome screen)
 7. terminal: tmux send-keys -t claude-pr42 Enter     # accept workspace trust
 8. terminal: tmux send-keys -t claude-pr42 \
      'Read /tmp/pr-42-feedback.txt. Apply each review point. Run pnpm test after each fix.' Enter
 9. loop:
      - terminal: tmux capture-pane -t claude-pr42 -p -S -50
      - (parse output, check for "idle" marker or error)
      - if error twice: break
10. terminal: cd /tmp/pr-42 && pnpm test
11. terminal: git -C /tmp/pr-42 commit -am "Apply PR review feedback"
12. terminal: git -C /tmp/pr-42 push
13. Reports back on Telegram with summary
```

Key details worth knowing:

- PR comments are fetched from GitHub at runtime via `gh pr view` — not from any file in the repo. Run `gh pr view --help` for available `--json` fields.
- Claude Code shows a workspace-trust dialog on first visit to a directory; the default (Enter) is correct.
- If you add `--dangerously-skip-permissions`, the default choice for the permissions-warning dialog is "No, exit" — Hermes's skill sends `Down` then `Enter` to accept. Don't paste `--dangerously-skip-permissions` outside a trusted repo.

---

## Expected Telegram Reply

```
✅ PR #42 review applied

Changes:
• Added Zod validation to POST /users and POST /orders
• Replaced 3 console.log calls with logger.info
• Added 4 tests in __tests__/user.test.ts (all passing)
• Added JSDoc to exported createUser()
• Renamed `tmp` → `userDraft` in 6 places

Tests: 47 passed, 0 failed
Pushed: abc1234 to origin/pr-42

Full diff: https://github.com/me/myproject/pull/42/files
```

---

## What Can Go Wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| "Claude Code hangs on permission prompt" | Auto-approve is off | Add `--dangerously-skip-permissions` to the delegation prompt (trusted repo only) |
| "Budget exceeded" | Task is larger than the model's `--max-budget-usd` | Raise the budget flag, or break into smaller delegations |
| "Tests never finish" | Test suite is slow; Hermes times out waiting | Set a longer `command_timeout` in `~/.hermes/config.yaml` |
| "Push fails: protected branch" | `pr-42` is protected | Push to a new branch `pr-42-review-applied` and open a follow-up PR |

---

## Variations

**Dry-run mode** — review only, don't push:
> "...apply the PR review comments. **Do not commit or push.** Just show me the diff when done."

**Bounded print mode** — for smaller tasks, skip tmux:
> "Use Claude Code in print mode with `--max-turns 15`. Task: add input validation to POST /users. Allowed tools: Read, Edit. Report back with the diff."

**Chained delegation** — review + deploy:
> "After applying the PR fixes and pushing, use Claude Code again in a fresh session to write the changelog entry based on the diff."

---

## Try This

Pick a real PR in one of your repos — ideally a small one with 2–3 comments. Run this exact flow end-to-end from your phone. Iterate on the prompt until the agent's behavior matches what you'd do manually.

Then save a working version of the prompt as a Hermes skill (`/skill create`) so the next time you just type `/pr-review-and-push 42` and it runs.
