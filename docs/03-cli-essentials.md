# 03 — CLI Essentials

> **Goal:** know every flag, shortcut, and config knob you'll actually use.
>
> **Official source:** [CLI Interface](https://hermes-agent.nousresearch.com/docs/user-guide/cli/) and [CLI Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands)

---

## The Command Families

Hermes is one binary with many subcommands. Grouped by what they do:

**Start/resume sessions**
```bash
hermes                         # interactive TUI
hermes -c                      # continue most recent session
hermes -r "session title"      # resume by title
hermes chat -q "prompt"        # single-query mode
hermes -s skill1,skill2        # load specific skills into this session
```

**Configure**
```bash
hermes setup                   # full wizard (provider + tools + gateway)
hermes model                   # pick LLM provider and model
hermes tools                   # enable/disable tool categories
hermes config set KEY VALUE    # set a single value
hermes config get KEY          # read a single value
hermes config check            # validate config
hermes config migrate          # update schema after an upgrade
```

**Authenticate (credential pools)**
```bash
hermes auth                    # interactive pool management wizard
hermes auth list               # show all credential pools and which is active
hermes auth list <provider>    # show a single provider's pool
hermes auth add <provider>     # add a credential (API key or OAuth)
hermes auth remove <provider> <index>
hermes auth reset <provider>   # clear cooldowns / exhaustion status
```

`hermes auth list` is the authoritative view for pool-based credentials (including OAuth tokens auto-imported from `~/.claude/.credentials.json`). `hermes status`'s "API Keys" section only surfaces `.env`-based key vars, so use `hermes auth list` if you're wondering whether a Claude Code login was picked up. Details in [chapter 1](01-installation.md#i-already-use-claude-code--do-i-need-a-separate-key).

**Messaging + scheduling**
```bash
hermes gateway setup           # configure Telegram/Discord/Slack/WhatsApp/etc.
hermes gateway                 # start the gateway process
hermes cron list               # show scheduled tasks
```

**Maintenance**
```bash
hermes update                  # upgrade to latest version
hermes doctor                  # diagnose problems
hermes status                  # one-shot config summary
hermes version                 # print version
```

**Migration**
```bash
hermes claw migrate            # import an existing OpenClaw setup
```

(OpenClaw was Hermes's predecessor — Nous Research's earlier autonomous-agent project. Only relevant if you were already running it; skip this command otherwise.)

---

## Terminal Backends

This is the setting that most new users don't know exists but should. Hermes can run its shell commands in six different places:

| Backend | Command runs on | Use when |
|---|---|---|
| `local` (default) | Your machine | Trusted projects |
| `docker` | Isolated container | Running untrusted/experimental code |
| `ssh` | Remote server | Hermes on laptop, work on a cloud VM |
| `daytona` | Daytona sandbox | Ephemeral dev environments |
| `singularity` | HPC cluster | Scientific computing |
| `modal` | Modal serverless | Pay-per-use, scales to zero |

Switch with:

```bash
hermes config set terminal.backend docker
```

The Docker backend drops all Linux capabilities and disables privilege escalation by default. That makes it the right choice when you're letting Hermes run arbitrary code from a wild prompt.

For SSH, you'll also set `terminal.ssh.host` and related keys — run `hermes config set` interactively and it walks you through them.

---

## Configuration File

All config lives in `~/.hermes/config.yaml`. You can edit it directly — changes take effect on the next session start. A minimal file looks like:

```yaml
provider:
  name: openrouter
  model: anthropic/claude-opus-4-7

terminal:
  backend: local
  command_timeout: 120

tools:
  enabled:
    - file
    - terminal
    - web
    - memory
```

Secrets (API keys) live in a separate `~/.hermes/.env` so you can commit `config.yaml` to dotfiles without leaking keys.

---

## Slash Commands (full list)

In-session commands start with `/`. Beyond the basics in chapter 2, useful ones are:

| Command | What it does |
|---|---|
| `/compress` | Summarize middle turns; preserve first 3 + last 4 |
| `/memory add "note"` | Force a fact into persistent memory |
| `/memory search "query"` | Recall past memories |
| `/skill list` | Show installed skills |
| `/skill create` | Interactive skill authoring |
| `/model` | Switch model mid-conversation |
| `/title "new name"` | Rename session |
| `/verbose` | Cycle tool output display |
| `/voice tts` | Have Hermes speak its replies |
| `/background <task>` | Spawn parallel agent |
| `/quit` | Exit cleanly |

Every installed skill also registers its own slash command — so if you install a skill called `github-pr-workflow`, you'll get `/github-pr-workflow` for free.

---

## Personalities & SOUL.md

Hermes has a concept of a **personality** — a system prompt layer that shapes tone and defaults. The built-in one lives in the repo and is applied automatically; you can override it per-session or per-project by creating a `SOUL.md` file.

```markdown
# ~/.hermes/SOUL.md

You are running as my research assistant.
Prefer concise answers. Never fabricate citations.
When unsure, run a web search rather than guess.
```

Per-project overrides go in `./SOUL.md` at the root of whatever directory you launch `hermes` from — it's loaded automatically.

---

## Context Management

With default tools and `SOUL.md`, expect roughly **6–8K input tokens per CLI turn** and **15–20K per Telegram message** (the gateway adds framing). *These are ballpark figures from a [third-party tutorial](https://www.nxcode.io/resources/news/hermes-agent-tutorial-install-setup-first-agent-2026) — measure your own workloads for anything cost-critical.* Two levers to reduce this:

1. **`/compress`** — Summarizes old turns. Middle of the conversation gets condensed; the first 3 and last 4 turns are preserved intact.
2. **Narrower toolset** — `hermes tools` lets you disable categories you're not using. Fewer tools in the system prompt = fewer tokens per turn.

For very long sessions, `/compress` before each big task is a good habit.

---

## Keyboard Shortcuts (complete)

| Key | Action |
|---|---|
| `Enter` | Send |
| `Alt+Enter` / `Ctrl+J` | New line |
| `Ctrl+C` | Interrupt / second press exits |
| `Ctrl+V` | Paste image (vision) |
| `Ctrl+B` | Start/stop voice recording (requires `voice` extra) |
| `Ctrl+R` | Search history |
| `Up` / `Down` | Scroll previous messages |
| `Tab` (after `/`) | Autocomplete slash commands |

---

**Next →** [04 — Tools & Skills](04-tools-and-skills.md)
