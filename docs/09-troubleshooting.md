# 09 — Troubleshooting

> **Goal:** resolve the problems that stop people cold in the first week.

When in doubt, start with:

```bash
hermes doctor
```

It's the single best diagnostic — it walks through every dependency, config file, API key, and permission, and tells you exactly what's wrong.

---

## Install Problems

### `hermes: command not found`

The symlink wasn't added or `~/.local/bin` isn't on your PATH.

```bash
ls ~/.local/bin/hermes        # should exist
echo $PATH | tr ':' '\n' | grep local/bin   # should be listed
```

Fix:
```bash
ln -sf "$(pwd)/venv/bin/hermes" ~/.local/bin/hermes
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Installer fails partway through

Re-run it — it's idempotent. If it fails again at the same step, run the manual install (chapter 1) to see the actual error.

### Termux install: `.[all]` won't build

Expected. The `voice` extra pulls `faster-whisper` which has no Android wheels.

```bash
uv pip install -e ".[termux]"
```

### `Python 3.11 not found`

`uv` downloads it automatically, but only if invoked correctly:

```bash
uv venv venv --python 3.11
```

If this fails, check network access — `uv` fetches Python builds from `python.org` mirrors.

---

## Configuration Problems

### `API key not set`

```bash
hermes model              # interactive
# or
hermes config set OPENROUTER_API_KEY sk-or-v1-...
```

Check:
```bash
cat ~/.hermes/.env
```

### `Minimum context: 64K tokens`

Your selected model's context window is too small. Pick a bigger one:

```bash
hermes model     # and pick a model with ≥64K context
```

For local models (via Ollama, llama.cpp), bump the context in the model's config before selecting it.

### Config schema changed after an update

```bash
hermes config check
hermes config migrate
```

---

## Runtime Problems

### Session hangs on a tool call

Ctrl+C once to interrupt. If that doesn't work, Ctrl+C twice to kill. The session state is still saved — `hermes -c` will resume.

### "Permission denied" when Hermes tries to run a command

Two causes:

1. **File system permissions** — the user running Hermes doesn't have access. Fix the file perms.
2. **Hermes' own permission gate** — auto-approve is off and Hermes is asking you. Type `y` to allow once, or `always` to whitelist the pattern.

### Gateway stops receiving Telegram messages

Check the process is up:
```bash
systemctl status hermes-gateway
journalctl -u hermes-gateway -n 50
```

Common causes:
- Bot token rotated
- Rate limit hit (Telegram will 429)
- Network interruption — the gateway auto-reconnects but logs the gap

### Claude Code sub-agent hangs

If you delegated to Claude Code (chapter 8) and it seems stuck:

1. In a separate terminal: `tmux ls` — see if the session is still there
2. `tmux attach -t claude-<name>` — see what Claude Code is actually doing
3. If it's waiting on the permission dialog, send `Down` then `Enter`
4. If it's mid-edit on a large file, just wait

For print-mode runs, there's no tmux session — check `~/.claude/logs/` for the most recent run.

---

## Cost Problems

### Tokens burning too fast

Diagnosis:
```bash
/compress        # reduce current session size
```

Long-term fixes:

1. Switch to a cheaper model: `hermes model` → pick a smaller OpenRouter model
2. Disable tool categories you don't use: `hermes tools`
3. Trim `SOUL.md` — every word there costs per-turn tokens
4. Fewer skills loaded globally; use `hermes -s skill1,skill2` to load per-session

### Telegram gateway burns 2× more tokens than CLI

Expected — see [chapter 5](05-memory-and-context.md#cost--context-window-realities). The gateway adds framing. If it matters, use a cheaper model specifically for gateway sessions:

```yaml
# ~/.hermes/config.yaml
gateway:
  provider: openrouter
  model: openai/gpt-4o-mini    # cheaper than your CLI model
```

---

## Backup and Uninstall

### Back up your Hermes state

Everything Hermes knows about you lives in `~/.hermes/`. Before a risky upgrade, OS reinstall, or machine migration, tar it up:

```bash
tar czf hermes-backup-$(date +%Y%m%d).tar.gz -C ~ .hermes
```

Restore on a new machine by extracting into `~` and re-running the installer to recreate the binary symlink:

```bash
tar xzf hermes-backup-YYYYMMDD.tar.gz -C ~
# then re-run the installer from chapter 1 to restore the `hermes` command
```

Your sessions, memories, skills, and cron jobs will all carry over. API keys in `~/.hermes/.env` carry over too — if that's a security concern for cross-machine moves, strip the keys before archiving and re-add them after.

### Uninstall

Complete removal:

```bash
# 1. Stop the gateway if it was installed as a systemd unit
sudo systemctl disable --now hermes-gateway 2>/dev/null
rm -f /etc/systemd/system/hermes-gateway.service

# 2. Remove the symlink
rm -f ~/.local/bin/hermes

# 3. Remove all state AND the code checkout
#    The installer defaults to placing the checkout at ~/.hermes/hermes-agent
#    (source: scripts/install.sh), so this single rm covers both data and code.
#    BACK THIS UP FIRST if you might come back — see the backup section above.
rm -rf ~/.hermes

# 4. If you passed --dir to the installer to put the checkout elsewhere,
#    remove that path too:
# rm -rf /your/custom/path
```

Step 3 is the destructive one. If you want to keep your memories and skills for a future reinstall, archive `~/.hermes/memories/` and `~/.hermes/skills/` before deleting.

---

## "It's Doing Weird Things"

Behavior that looks like a bug but usually isn't:

### Hermes is using a tool I didn't expect

It has wide autonomy by default. Scope it with a more specific prompt:

> "Use only the `read_file` and `web_search` tools for this. Do not run shell commands."

Or tighten the global config:
```bash
hermes tools      # disable the tool category entirely
```

### Memories aren't being recalled

1. Check they actually exist: `ls ~/.hermes/memories/`
2. Search manually: `/memory search "keyword"`
3. If the search returns nothing, the memory may not have been written — try `/memory add "..."` explicitly

### Same prompt, different results

This is an LLM, not a function. Variability is baseline. If you need determinism:

- Set the provider's temperature to 0 (`hermes config set provider.temperature 0`)
- Provide more context in the prompt
- Capture successful patterns into skills

---

## Getting Help

In order of usefulness:

1. `hermes doctor` — 80% of problems are diagnosed here
2. `hermes status` — compact setup summary, copy-pasteable into issues
3. [GitHub Discussions](https://github.com/NousResearch/hermes-agent/discussions)
4. [Nous Research Discord](https://discord.gg/NousResearch) — active community
5. [GitHub Issues](https://github.com/NousResearch/hermes-agent/issues) — for confirmed bugs

When asking for help, paste the output of `hermes status` and `hermes doctor`. It saves the maintainers (and you) an entire back-and-forth.

---

## Where to Go Next

You've finished the core tutorial. From here:

- **Learn by building** — pick a real workflow you do weekly and teach Hermes to do it. Let the auto-skill loop turn it into a reusable skill.
- **Connect it to your stack** — MCP servers for your company's tools, scheduled reports, pager integrations.
- **Read the source** — the codebase is small enough to understand end-to-end. Start with `hermes/agent/loop.py`.
- **Contribute a skill** — agentskills.io accepts community submissions.

See [`exercises/README.md`](../exercises/README.md) for progressively harder challenges.

---

← [Back to README](../README.md)
