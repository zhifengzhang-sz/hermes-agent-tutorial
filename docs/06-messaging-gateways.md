# 06 — Messaging Gateways

> **Goal:** reach Hermes from Telegram (or Discord, Slack, WhatsApp, Signal, Email) while it runs on your server.

---

## Why Gateways Matter

The whole point of Hermes living on a VPS rather than your laptop is that you can talk to it from anywhere. Gateways are the bridge — one process that listens on a messaging platform, forwards messages into a Hermes session, and pipes the agent's replies back out.

Supported platforms out of the box:

- **Telegram** — easiest to set up, best-polished UX
- **Discord** — works in DMs and servers
- **Slack** — for team agents
- **WhatsApp** — via Baileys bridge (Node deps required)
- **Signal** — via `signal-cli`
- **Email** — for async / batch workflows
- **SMS** — via an SMS gateway service (`sms` extra)
- **Home Assistant** — for smart-home triggers (if you run Home Assistant on your network, Hermes can call services and read entity states as native tools, and can be reached via HA automations; configuration is outside this tutorial's scope — see the [official Home Assistant docs for Hermes](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/homeassistant) for setup)
- **Additional specialized platforms** — DingTalk, Feishu, WeCom (+ callback variant), Weixin, BlueBubbles, QQBot. Each ships as its own pip extra (e.g., `dingtalk`, `feishu`); see the [official gateway docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) for the full matrix and setup instructions.

All of them run under a single `hermes gateway` process — one command, many surfaces.

---

## Setup (Telegram as the example)

Install the messaging extra if you didn't use `.[all]`:

```bash
uv pip install -e ".[messaging]"
```

Then run the wizard:

```bash
hermes gateway setup
```

It asks which platforms you want. For Telegram, you'll need:

1. A **bot token** from [@BotFather](https://t.me/BotFather) on Telegram
2. Your **user ID** (get it from [@userinfobot](https://t.me/userinfobot))

Paste both when prompted. The wizard writes them to `~/.hermes/.env`:

```bash
TELEGRAM_BOT_TOKEN=1234567890:AA...
TELEGRAM_ALLOWED_USER_IDS=123456789
```

The `ALLOWED_USER_IDS` list is a hard allowlist. The bot ignores messages from anyone not on it. **Do not skip this** — an open Telegram bot with code execution is a very bad idea.

---

## Start the Gateway

```bash
hermes gateway
```

Process runs in the foreground. On your server you probably want it under systemd, `tmux`, or a process manager. Minimal systemd unit:

```ini
# /etc/systemd/system/hermes-gateway.service
[Unit]
Description=Hermes Agent messaging gateway
After=network-online.target

[Service]
Type=simple
User=hermes
WorkingDirectory=/home/hermes/hermes-agent
ExecStart=/home/hermes/.local/bin/hermes gateway
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable:
```bash
sudo systemctl enable --now hermes-gateway
journalctl -u hermes-gateway -f    # watch logs
```

---

## Sending Your First Message

Open Telegram, message your bot: `What's the load average on this server?`

You should see a streaming reply with `💻 uptime` tool use and a natural-language summary.

---

## Auto-Approve Mode

By default, if Hermes hits a permission gate mid-task (write a file, run a command with side effects), it pauses and asks for confirmation. From Telegram, you can't grant that — so the task stalls.

Enable auto-approve during gateway setup (or later):

```bash
hermes config set gateway.auto_approve true
```

Trade-off: the agent can now do anything its toolset allows. Only enable this on **trusted projects** where you're comfortable with that blast radius. For destructive work, either disable auto-approve or switch the terminal backend to `docker` so commands run in an isolated container.

---

## Discord, Slack, WhatsApp

Same wizard, different credentials:

- **Discord** — needs a bot token from the [Developer Portal](https://discord.com/developers/applications) and (optionally) a server ID
- **Slack** — needs a Slack app with `chat:write` and `im:history` scopes, plus bot and signing tokens
- **WhatsApp** — needs Node.js installed (`npm install` from your `hermes-agent` checkout, which pulls the Baileys bridge deps) and a QR pairing flow (scan from your phone)
- **Signal** — needs `signal-cli` installed and a registered phone number

All credentials land in `~/.hermes/.env`. The gateway process picks up any platform with valid credentials automatically — no separate start commands needed.

---

## Voice Mode (bonus)

If you installed the `voice` extra:

```bash
uv pip install -e ".[voice]"
```

…then in the CLI, press `Ctrl+B` to record, release to transcribe (via local `faster-whisper`) and send.

In messaging, voice notes sent to the bot get transcribed automatically. To have Hermes **reply** with voice, use `/voice tts` in the CLI or send `/voice tts` in Telegram. Voices come from Piper (free, local) or ElevenLabs (premium, cloud) depending on which extras you installed.

---

## Security Checklist

Before you put a gateway on the public internet:

- [ ] `TELEGRAM_ALLOWED_USER_IDS` set (or the equivalent on your platform)
- [ ] Terminal backend set to `docker` for untrusted work
- [ ] `~/.hermes/.env` has `chmod 600`
- [ ] Run the gateway as a non-root user
- [ ] Use a model you trust for tool use — prompt injection via a stray email or scraped page is a real risk
- [ ] Review `~/.hermes/logs/` regularly for unexpected tool calls

---

## Try This

Set up the Telegram gateway and test three requests from your phone:

1. "What's the disk usage on this server?"
2. "Summarize the top 5 Hacker News stories right now."
3. "Write a haiku about the current CPU load."

The third one is a sanity check — it should produce a haiku that actually references real data, not a generic one.

See also: [`examples/telegram-morning-briefing/`](../examples/telegram-morning-briefing/) for a scheduled Telegram task.

---

**Next →** [07 — Scheduled Tasks](07-scheduled-tasks.md)
