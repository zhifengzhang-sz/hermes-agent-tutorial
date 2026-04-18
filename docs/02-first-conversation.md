# 02 — Your First Conversation

> **Goal:** talk to Hermes, watch it use tools, and understand what just happened.
>
> **Official source:** [Quickstart](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart)

---

## Start a Session

```bash
hermes
```

You'll see a TUI prompt. Type a question that forces Hermes to *do* something, not just chat:

```
> What's in my current directory? List files and tell me what this project does.
```

Hermes will stream back tool calls in real time — you'll see something like `💻 ls -la`, then `📄 reading README.md`, then a synthesized answer. This is the **agentic loop**: the model decides which tool to call, Hermes executes it, the output feeds back into the next turn.

---

## What Just Happened

Hermes ships with several dozen built-in tools. The ones you'll see on day one:

- **File operations** — `read`, `write`, `search`, `edit`
- **Terminal** — runs shell commands and captures output
- **Web** — search, scrape, fetch
- **Memory** — writes notes to `~/.hermes/memories/`
- **Time** — the agent knows what day it is

The model picks the tool. You don't have to tell it "run `ls`" — you say "what's in this directory?" and it figures out the rest.

---

## Useful Keystrokes

These aren't in the welcome banner, but you'll want them immediately:

| Key | Does |
|---|---|
| `Enter` | Send message |
| `Alt+Enter` or `Ctrl+J` | New line (multiline input) |
| `Ctrl+V` | Paste an image from clipboard (vision) |
| `Ctrl+C` | Interrupt the current task |
| `Tab` (after `/`) | Autocomplete slash commands |
| `Ctrl+B` | Start/stop voice recording (requires `voice` extra) |

The interrupt behavior is the one most people miss: **you don't have to wait for Hermes to finish**. Type a new message and press Enter — it cancels the current task and switches to your new instructions mid-turn.

---

## Slash Commands You'll Use Today

Type `/` and press Tab to see all of them. The essentials:

| Command | What it does |
|---|---|
| `/compress` | Summarize conversation history to free up context |
| `/model` | Switch LLM provider or model mid-session |
| `/title` | Rename the session |
| `/verbose` | Cycle tool-output verbosity: off → new → all → verbose |
| `/background <prompt>` | Spawn an isolated parallel session |
| `/quit` | Exit |

`/background` is sneaky-useful. It spawns a *separate* agent session in a daemon thread, inherits your current model and tool config, but gets a fresh conversation history. You can kick off "analyze every log file in `/var/log` and summarize today's errors" and keep chatting in the foreground while it works.

---

## Resuming Sessions

Close your terminal, come back tomorrow, and:

```bash
hermes -c                    # continue the most recent session
hermes -r "my research"      # resume by title (fuzzy match)
```

Your full conversation history is restored — the model picks up where you left off.

---

## Single-Query Mode

For scripts or one-shot tasks, skip the TUI entirely:

```bash
hermes chat -q "Summarize today's changes in ~/projects/my-repo using git log"
```

This runs the task, prints the answer, and exits. Pipe-friendly.

---

## Session-Scoped Skills

You can load specific skills into just one session without turning them on globally:

```bash
hermes -s github-pr-workflow,github-auth
```

Each named skill is merged into the system prompt before the first turn. This is how you keep the agent focused — you don't want every skill available for every task.

---

## Try This

Three small challenges to get a feel for how the agent thinks:

1. **Ask it to find something factual:** "What's the current weather in Tokyo?" — watch it pick the web tool.
2. **Ask it to modify a file:** in a throwaway directory, say "create a Python script that prints fibonacci(10) and run it."
3. **Give it a vague goal:** "Look at the git history of this repo and tell me who the top 3 contributors are." — notice how it plans a sequence of tool calls.

When you're comfortable with all three, move on.

---

**Next →** [03 — CLI Essentials](03-cli-essentials.md)
