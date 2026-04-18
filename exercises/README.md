# Exercises

Progressively harder challenges to cement what you've learned. Do them in order — each one builds on capabilities from the previous.

Each exercise lists:
- **Goal** — what you're building
- **Success criteria** — how you know you're done
- **Hints** — where to look in the tutorial
- **Stretch** — optional extension

---

## 🟢 Level 1 — Fundamentals

### Exercise 1.1 — Prove It's Alive

**Goal:** confirm your install works end-to-end.

**Success criteria:**
- `hermes doctor` reports no issues
- `hermes chat -q "What's 17 * 23?"` returns `391`
- You can start, interrupt (Ctrl+C), and resume (`hermes -c`) a session

**Hints:** [chapter 1](../docs/01-installation.md), [chapter 2](../docs/02-first-conversation.md)

---

### Exercise 1.2 — Your Filesystem, Explained

**Goal:** have Hermes describe a directory you care about.

In a session, ask: *"Look at `~/code/<some-project>`, tell me what language it's in, what build system it uses, and list the top 3 entry points."*

**Success criteria:** the answer is accurate. If it's not, figure out why — was it missing a tool? Was the prompt too vague?

**Stretch:** make it work for any directory you pass as an argument via `hermes chat -q`.

---

### Exercise 1.3 — Vision Test

**Goal:** use the image paste feature.

Take a screenshot of any GitHub README. In a Hermes session, `Ctrl+V` to paste, then ask: *"Summarize this project and list its dependencies."*

**Success criteria:** Hermes reads the image and produces a coherent summary.

**Hints:** [chapter 3](../docs/03-cli-essentials.md#keyboard-shortcuts-complete)

---

## 🟡 Level 2 — Skills & Memory

### Exercise 2.1 — First Custom Skill

**Goal:** write a skill that drafts your daily standup.

Start from [`examples/hello-skill/`](hello-skill/) and modify. The skill should:
- Read `git log --since=yesterday --author="$(git config user.name)"` across a configured list of repos
- Produce 3 bullets: what got done, what's in progress, blockers
- Output nothing else — no preamble, no closing

**Success criteria:** `/standup-draft` produces a bullet list you'd actually post in Slack.

**Hints:** [chapter 4](../docs/04-tools-and-skills.md#writing-your-first-skill)

**Stretch:** have the skill prompt you for blockers if it can't infer them from commits.

---

### Exercise 2.2 — Preference Memory

**Goal:** teach Hermes something about you that persists across sessions.

In a session, say: *"I prefer TypeScript over JavaScript for all new web projects. Remember this."*

Quit. Start a fresh session. Ask: *"Scaffold a new API for me."* Verify Hermes picks TypeScript without being told again.

**Success criteria:** behavior changes across sessions.

**Hints:** [chapter 5](../docs/05-memory-and-context.md#the-auto-memory-loop)

**If it fails:** check `~/.hermes/memories/`. If the preference wasn't written, force it: `/memory add "I prefer TypeScript for all new web projects"`.

---

### Exercise 2.3 — Audit the Auto-Skill Loop

**Goal:** observe the skill-creation nudge fire in a real session and see what (if anything) the agent decides to write. Before starting, skim [chapter 4's "Learning Loop"](../docs/04-tools-and-skills.md#the-learning-loop--what-it-actually-is) so you're evaluating the right thing — it's a periodic prompt injection, not actual model learning.

Turn down the nudge interval so you don't have to sit through the default of 10 turns. In `~/.hermes/config.yaml`:

```yaml
skills:
  creation_nudge_interval: 3
```

Then walk through the same workflow three times in separate sessions. Examples:
- Three times: "set up a new Vite + React + Tailwind project in `~/tmp/test-N`"
- Three times: "deploy the current branch to staging"

After a few turns past the third run, the model should see a nudge prompt and may (or may not) call the `skill_manager` tool to write a SKILL.md. Check `~/.hermes/skills/` for new folders.

**Success criteria:** a new skill file appears, OR you observe the nudge fire in the session transcript and the model explicitly decide against writing one.

**Things to notice:** the nudge is LLM-driven. A weaker model may skip the reminder or produce a thin skill. A stronger model may write something useful. Either way, revert `creation_nudge_interval` to something saner (10 or higher) after this exercise.

**If no nudge fires at all:** force skill creation with `/skill create` and walk through the wizard based on the current session.

---

## 🟠 Level 3 — Remote Access

### Exercise 3.1 — Telegram In Under 10 Minutes

**Goal:** message Hermes from your phone.

**Success criteria:** send "what's the load average?" from Telegram and get a real number back.

**Hints:** [chapter 6](../docs/06-messaging-gateways.md)

---

### Exercise 3.2 — Production the Gateway

**Goal:** the gateway survives a reboot.

- Write a systemd unit file (copy from [chapter 6](../docs/06-messaging-gateways.md#start-the-gateway))
- `sudo systemctl enable --now hermes-gateway`
- Reboot the machine
- Verify Telegram still works without any intervention

**Success criteria:** you can reboot and the agent is reachable from Telegram within 60 seconds.

**Stretch:** add a `OnFailure=` email or ntfy notification so you know if it crashes.

---

### Exercise 3.3 — Locked-Down Terminal Backend

**Goal:** run Hermes' shell commands in Docker instead of directly on the host.

```bash
hermes config set terminal.backend docker
```

Then ask Hermes to identify its runtime: *"Run `hostname`, then `whoami`, then `cat /etc/os-release`, and tell me what machine you're on."*

Run the same three commands in a second terminal on the host. Compare. If the backend switch took effect, Hermes's output **should not match** the host's — at minimum, `hostname` should differ. A second check: *"Create a file called `/tmp/container-proof-$(date +%s)` and then run `ls /tmp/` and tell me what's there."* Then run `ls /tmp/container-proof-*` **on the host**. If the file doesn't appear on the host, Hermes wrote it somewhere the host can't see — i.e., inside whatever container the Docker backend launched.

**Success criteria:** the identity Hermes reports doesn't match the host, and the proof file doesn't show up on the host filesystem.

**Hints:** [chapter 3](../docs/03-cli-essentials.md#terminal-backends)

**Do not** test isolation by asking Hermes to delete things. If the backend switch silently failed, a destructive test would hit your actual filesystem. Stick to read-only and marker-file checks.

---

## 🔴 Level 4 — Orchestration

### Exercise 4.1 — First Scheduled Task

**Goal:** have Hermes message you something scheduled.

Pick anything simple:
- "Every day at noon, tell me a joke on Telegram."
- "Every Monday at 9am, send me a quote about discipline."

**Success criteria:** the message arrives at the scheduled time without you doing anything.

**Hints:** [chapter 7](../docs/07-scheduled-tasks.md)

---

### Exercise 4.2 — Morning Briefing

**Goal:** set up [`examples/telegram-morning-briefing/`](telegram-morning-briefing/) for real.

Customize it:
- Your actual city for weather
- Your actual repo list
- Your real priorities for the "email" section (or skip if no Gmail MCP)

Run it for a full week. Each day, tune the prompt based on how useful the briefing was.

**Success criteria:** by Friday, the briefing is something you actually read instead of dismiss.

---

### Exercise 4.3 — Delegate to Claude Code

**Goal:** complete one real coding task via delegation from Telegram.

Pick a small, low-stakes task in a throwaway repo:
- Add a new REST endpoint
- Refactor one file to use a different style
- Write tests for an existing function

Send the delegation prompt from Telegram. Let Hermes orchestrate Claude Code. Don't touch your laptop.

**Success criteria:** the task is complete, tests pass, changes are pushed — and you did it all from your phone.

**Hints:** [chapter 8](../docs/08-claude-code-integration.md), [`examples/claude-code-delegation/`](claude-code-delegation/)

**Stretch:** wrap the successful prompt pattern into a reusable skill.

---

## 🟣 Level 5 — Make It Yours

### Exercise 5.1 — A Skill Library

**Goal:** build 5 skills that reflect *your* workflow.

Ideas:
- `code-review` — your review checklist, applied to a diff or PR
- `meeting-notes` — take raw notes, produce action items and a Slack summary
- `research-recap` — read 5 URLs and write a 200-word synthesis
- `incident-triage` — pull in logs, metrics, and recent deploys; produce a status message
- `weekly-review` — Friday reflection prompts + memory-writing

Each skill < 40 lines of markdown.

**Success criteria:** you use at least 3 of them weekly without thinking about it.

---

### Exercise 5.2 — MCP Integration

**Goal:** plug an MCP server into Hermes and use it in a real task.

Options:
- GitHub MCP — manage issues and PRs from chat
- Linear MCP — query and update tickets
- Your own MCP server — expose an internal API or database

**Success criteria:** a real task you used to do manually is now one Telegram message.

**Hints:** [chapter 4](../docs/04-tools-and-skills.md#mcp-servers)

---

### Exercise 5.3 — Contribute Back

**Goal:** publish one of your skills.

If any skill from 5.1 is generalizable, submit it to [agentskills.io](https://agentskills.io) or open a PR against [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent).

**Success criteria:** someone else uses your skill.

---

## Done?

You've gone from zero to running a production-ish Hermes deployment with skills, scheduling, remote access, and sub-agent delegation. The whole thing runs on your hardware, respects your privacy, and gets more capable the longer you use it.

What's next is entirely up to what you want automated. Go build something only you would build.

---

← [Back to README](../README.md)
