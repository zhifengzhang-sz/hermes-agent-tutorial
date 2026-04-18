# 01 — Installation

> **Goal:** get `hermes` on your PATH and running, in under 5 minutes.
>
> **Official source:** [hermes-agent.nousresearch.com/docs/getting-started/installation](https://hermes-agent.nousresearch.com/docs/getting-started/installation)

---

## Before You Start

Hermes is a local CLI that talks to a remote LLM. **You** supply the LLM credential; Nous Research does not issue a "Hermes API key," and there is no Hermes cloud you're logging into. Before running the installer, have two things ready:

1. A Unix-like shell (Linux, macOS, WSL2, or Termux on Android) with `git` installed — check with `git --version`.
2. **One LLM provider credential.** Hermes has to call a model somewhere. Pick one:

| Provider | Best if | What to grab before starting |
|---|---|---|
| **Claude Code (reuse)** | You already use Claude Code AND you're willing to fund (or have funded) "Extra usage" credit at claude.ai/settings/usage | Hermes auto-imports `~/.claude/.credentials.json`, but Anthropic bills third-party app calls against *Extra usage*, not your subscription plan. See the "I already use Claude Code" note below for the full picture |
| [**OpenRouter**](https://openrouter.ai) | You don't already have a model provider account | Sign up, add a starting balance (see OpenRouter's pricing page to estimate how much for your expected use), copy the `sk-or-v1-...` key |
| **Anthropic API** | You have a standalone `sk-ant-...` key (not just Claude Code) | Your existing `sk-ant-...` key |
| **Nous Portal** | You want Nous Research's own hosted models | Nothing to grab now — you'll OAuth in after install via `hermes model` |
| **OpenAI / Gemini / local Ollama / others** | You already have these set up | Your existing key, or the local endpoint URL |

You only need **one** of these. The Claude Code reuse row is zero-**config**, but it's not zero-**cost** — read the note below before assuming it's free. OpenRouter is the widest model selection for the least total setup if you're starting from scratch.

### "I already use Claude Code — do I need a separate key?"

**Hermes imports your Claude Code login automatically, but using it to drive Hermes turns isn't free.** The import part works; the billing part has a catch.

**What gets imported.** On startup, Hermes auto-discovers credentials at `~/.claude/.credentials.json` (the file `claude` writes when you log in) and seeds them as an Anthropic entry in its credential pool with `source: claude_code`. Verify:

```bash
hermes auth list
```

A working import looks like:

```
anthropic (1 credentials):
  #1  claude_code          oauth   claude_code ←
```

**The billing catch.** As of 2026-04, Anthropic's policy routes third-party apps (which includes Hermes) against your **"Extra usage"** credit bucket at [claude.ai/settings/usage](https://claude.ai/settings/usage) — **not** your Max/Pro plan limits. Third-party apps don't consume subscription quota. If that Extra-usage bucket is at zero, the first Hermes call fails with:

```
HTTP 400: Third-party apps now draw from your extra usage, not your plan limits.
          Add more at claude.ai/settings/usage and keep going.
```

The credential is valid; the balance isn't funded. Three options from there:

1. **Fund Extra usage** at claude.ai/settings/usage — lets you keep a single Anthropic login, but you're paying for Hermes turns separately from whatever your subscription covers.
2. **Get a standalone Anthropic API key** at [console.anthropic.com](https://console.anthropic.com) and put it in `~/.hermes/.env` as `ANTHROPIC_API_KEY=sk-ant-...`. Billed pay-per-token against the API key, not against your subscription. When both credentials exist in the pool, Hermes picks one based on the rotation strategy (see `credential_pool_strategies` in the credential-pools doc).
3. **Use a different provider** entirely (OpenRouter, OpenAI, Gemini, local Ollama) — keeps your Claude budget exclusively for Claude Code's own use.

A common setup given the above: run Hermes on a non-Anthropic model via OpenRouter or OpenAI (cheaper per turn for the bookkeeping-style work Hermes does), and let Hermes **delegate heavy coding tasks to Claude Code** as a sub-agent ([chapter 8](08-claude-code-integration.md)). Your Claude subscription still backs Claude Code — the subprocess, not Hermes — and Hermes' own usage is on a different bill.

> ⚠️ **`hermes status` is misleading for pool-based OAuth.** Its "API Keys" section only reports `.env`-based key vars; its "Auth Providers" section only lists named login providers (Nous Portal, Codex, Qwen). A Claude Code OAuth import will show as `Anthropic ✗ (not set)` in `hermes status` even when the credential is present and active. `hermes auth list` is the authoritative view.

Sources:
- Credential-pool auto-import behavior: `website/docs/user-guide/features/credential-pools.md` in the Hermes repo.
- Third-party app billing policy: verified live via an HTTP 400 from `api.anthropic.com` on 2026-04-18; the error body points to `claude.ai/settings/usage`. Anthropic's policy may change — if this note looks out of date, trust the actual error message your install produces.

### What it'll cost you

Hermes itself adds nothing to your bill — your cost is entirely whatever the LLM provider you pick charges per token. A few directional notes:

- **OpenRouter / Anthropic / OpenAI** are all pay-per-token; check the provider's current pricing page before running a long session, and prefer smaller models for routine work. Per-turn token budgets for Hermes itself are discussed in [chapter 5](05-memory-and-context.md#cost--context-window-realities), though the dollar-per-token figure is yours to look up.
- **Claude Code reuse** imports the credential but **does not** bill against your Max/Pro subscription — per Anthropic's current policy, third-party apps draw from your "Extra usage" balance at claude.ai/settings/usage. Plan on funding that bucket (or using a separate API key / provider) before your first Hermes turn. See the "I already use Claude Code" note above for detail.
- **Local Ollama / llama.cpp** is free after hardware, but needs a model with ≥64K context window (see [chapter 9](09-troubleshooting.md)).

Copy the key to your clipboard (or have your Nous Portal / Claude Code login handy), then continue.

---

## The One-Line Installer (recommended)

On Linux, macOS, WSL2, or Android/Termux:

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

The installer handles everything: `uv`, Python 3.11, Node.js 22, `ripgrep`, `ffmpeg`, the repo clone, the virtualenv, the global `hermes` symlink, and a first-run LLM provider prompt. The only thing it expects you to have is `git`.

**Windows users:** native Windows is not supported. Install [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) first, then run the command above inside the WSL2 shell.

**Termux users:** the installer auto-detects Termux and uses a curated `.[termux]` extra (no `voice`, since `faster-whisper` has no Android wheels). For the fully manual path, see the [Termux guide](https://hermes-agent.nousresearch.com/docs/getting-started/termux).

After it finishes, reload your shell:

```bash
source ~/.bashrc    # or source ~/.zshrc
```

Then type `hermes` — you should land in a chat prompt.

---

## Manual Installation (full control)

Skip this unless you need to pin versions, audit the install, or run in a locked-down environment.

```bash
# 1. uv (fast Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Clone with submodules
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# 3. Create a Python 3.11 venv (uv fetches it if missing)
uv venv venv --python 3.11
export VIRTUAL_ENV="$(pwd)/venv"

# 4. Install with every extra enabled
uv pip install -e ".[all]"

# 5. (optional) Node deps for browser + WhatsApp bridge
npm install

# 6. Config directory + empty .env
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills,pairing,hooks,image_cache,audio_cache,whatsapp/session}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env
chmod 600 ~/.hermes/.env    # the file holds API keys — keep it owner-only

# 7. Add at least one API key
echo 'OPENROUTER_API_KEY=sk-or-v1-your-key' >> ~/.hermes/.env

# 8. Global symlink
mkdir -p ~/.local/bin
ln -sf "$(pwd)/venv/bin/hermes" ~/.local/bin/hermes

# 9. Verify
hermes doctor
```

If `~/.local/bin` isn't on your `PATH`, add it:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

---

## Optional Extras

The `.[all]` bundle enables everything. If you want a leaner install, pick from:

| Extra | Adds | When to use |
|---|---|---|
| `messaging` | Telegram, Discord, Slack gateway | You want phone access |
| `cron` | Scheduled tasks parser | Morning briefings, backups |
| `voice` | Mic input + `faster-whisper` STT | Hands-free CLI (not on Termux) |
| `tts-premium` | ElevenLabs voices | Spoken replies |
| `mcp` | Model Context Protocol | Connecting Hermes to external MCP servers |
| `honcho` | AI-native memory | Deeper user modeling |
| `pty` | PTY terminal support | Interactive TUI sub-processes like `claude` |
| `modal` | Modal cloud backend | Running shell commands serverlessly |
| `dev` | `pytest` + test utilities | Contributing to Hermes itself |

Combine them: `uv pip install -e ".[messaging,cron,mcp]"`.

For the full list, see the [official extras table](https://hermes-agent.nousresearch.com/docs/getting-started/installation#manual-installation).

---

## Verify the Install

Run these, in order:

```bash
hermes version    # prints the version
hermes doctor     # diagnoses missing deps, wrong Python, bad config
hermes status     # shows configured provider, enabled tools
hermes chat -q "Hello! What tools do you have?"
```

If `hermes chat -q` streams back a coherent answer that references file ops, terminal, and web tools, **you're done**. Move on to [02 — Your First Conversation](02-first-conversation.md).

---

## Things That Go Wrong (and Fixes)

| Symptom | Fix |
|---|---|
| `hermes: command not found` | `source ~/.bashrc`, or check `echo $PATH` includes `~/.local/bin` |
| `API key not set` | `hermes model` (interactive) or `hermes config set OPENROUTER_API_KEY sk-or-...` |
| Stale config after an update | `hermes config check && hermes config migrate` |
| Termux install fails on `.[all]` | Use `.[termux]` — the `voice` extra has no Android wheels |
| `Minimum context: 64K tokens` | Your local model is too small; pick a different model with `hermes model` |

For anything else, run `hermes doctor` — its output is specifically designed to be copy-pasted into Discord or a GitHub issue.

---

**Next →** [02 — Your First Conversation](02-first-conversation.md)
