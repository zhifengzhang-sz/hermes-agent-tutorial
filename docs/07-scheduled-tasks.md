# 07 — Scheduled Tasks

> **Goal:** have Hermes do things unattended, on a schedule, and deliver results to you via a messaging gateway.

---

## The Pitch

Natural-language cron. Instead of writing:

```
0 9 * * * /usr/local/bin/fetch-hn.sh | mail -s "HN Digest" me@example.com
```

…you say:

```
Every morning at 9am, check Hacker News for AI news and send me a summary on Telegram.
```

Hermes parses the schedule, stores the task in `~/.hermes/cron/jobs.json` (one file, all jobs), and the gateway process runs it at the right time. The result is delivered through whichever messaging platform you configured.

---

## Prerequisites

```bash
uv pip install -e ".[cron,messaging]"
```

And you need at least one gateway set up (see [chapter 6](06-messaging-gateways.md)) — otherwise Hermes has nowhere to send the results.

The gateway process must be running for scheduled tasks to fire. Running tasks only at session-start isn't supported; they truly fire from the daemon.

---

## Creating a Task

Easiest way: just tell the agent.

```
> Every weekday at 8am, check my inbox for anything marked urgent and send me a summary on Telegram. Skip weekends.
```

Hermes responds with something like:

```
✓ Scheduled task created: "urgent-inbox-summary"
  Schedule: Mon-Fri at 08:00 (cron: 0 8 * * 1-5)
  Next run: tomorrow at 08:00
  Delivery: Telegram
```

Under the hood it appended an entry to `~/.hermes/cron/jobs.json`. The shape of one job is roughly:

```json
{
  "name": "urgent-inbox-summary",
  "schedule": "0 8 * * 1-5",
  "prompt": "Check my inbox for anything marked urgent since yesterday at 8am.\nSummarize in 3 bullets or fewer. If there's nothing urgent, say so.",
  "deliver": "telegram",
  "model": null,
  "provider": null
}
```

Key things to notice:

- It's one JSON file per Hermes user, not one file per task.
- `deliver` is a flat string (`"telegram"`, `"local"`, `"origin"`, etc.), not a nested object with `channel` and `user_id` — the channel's recipient (your Telegram user ID, etc.) is picked up from `~/.hermes/.env`, so the cron job doesn't restate it.
- `model` and `provider` are top-level scalar overrides. Set them if you want this specific job to run on a different model than your global config; leave them `null` to inherit.

The preferred way to manage jobs is through `hermes cron` commands or by asking the agent in natural language — both below. Hand-editing the JSON works too; the scheduler re-reads `jobs.json` on every tick (source: `cron/jobs.py` → `get_due_jobs()` → `load_jobs()`), so changes take effect on the next tick without restarting the gateway. Do back up first — if your edit produces invalid JSON, the scheduler surfaces an error rather than silently skipping the bad entry.

Source: `cron/jobs.py` in the Hermes repo.

---

## Managing Tasks

```bash
hermes cron list              # show all scheduled tasks
hermes cron show NAME         # inspect one
hermes cron disable NAME      # pause (without deleting)
hermes cron enable NAME       # resume
hermes cron delete NAME       # remove
hermes cron run NAME          # fire once, now, for testing
```

Or, inside a session:

```
/cron list
/cron run urgent-inbox-summary
```

---

## Schedule Syntax

Hermes accepts cron expressions *and* natural language. Examples it parses correctly:

| You say | Cron |
|---|---|
| "Every morning at 9" | `0 9 * * *` |
| "Every Monday at 10am" | `0 10 * * 1` |
| "Every hour" | `0 * * * *` |
| "Every 15 minutes during business hours" | `*/15 9-17 * * 1-5` |
| "The first of every month at noon" | `0 12 1 * *` |
| "Every Tuesday and Thursday at 7pm" | `0 19 * * 2,4` |

If the parse is ambiguous, Hermes asks for clarification before saving.

---

## Task Design Patterns

A few patterns that work well:

**The digest** — scrape a source, summarize, deliver.
> "Every day at 7am, read the top 5 Hacker News stories and summarize each in one sentence. Send on Telegram."

**The monitor** — check a state, alert on change.
> "Every 10 minutes, check if my production site is responding with 200. If not, ping me on Telegram."

**The routine check-in** — prompt you to do a thing.
> "Every Friday at 4pm, ask me what I want to focus on next week and save my answer as a memory."

**The cleaner** — run maintenance.
> "Every Sunday at 3am, delete any file older than 30 days in `/var/tmp/cache`."

For a worked example, see [`examples/telegram-morning-briefing/`](../examples/telegram-morning-briefing/).

---

## Gotchas

1. **Task output goes to your delivery channel, not back to a CLI session.** If you run `hermes cron run` manually, output prints to stdout; when the scheduler fires it, output is routed by the job's `deliver` field.
2. **Long tasks block the scheduler thread.** For tasks that take >1 minute, consider `/background` from the task prompt, or run them in Docker backend.
3. **Timezone is your server's local time.** Explicitly set `TZ` in the systemd unit or `.env` if the server is on UTC and you want Pacific mornings.
4. **The gateway must be running**, but missed runs don't accumulate. If a recurring job's scheduled time is more than one period in the past (e.g. an hourly job missed several hours), the scheduler fast-forwards it to the *next future* run instead of replaying the backlog. Only jobs whose due time sits inside the current period fire on the next tick (source: `cron/jobs.py` → `get_due_jobs()`).

---

## Try This

Set up a "daily standup draft" task:

> "Every weekday at 8:45am, look at my git commits from the previous 24 hours across all repos in ~/code, and draft a 3-bullet standup post. Send it to me on Telegram."

Verify with `hermes cron run daily-standup-draft`. Tune the prompt until the output is actually usable, then let it run on schedule.

---

**Next →** [08 — Claude Code Integration](08-claude-code-integration.md)
