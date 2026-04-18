# Example: Telegram Morning Briefing

A scheduled task that runs every weekday at 7am, gathers information from several sources, and sends you a compact briefing on Telegram.

This is the concrete version of the patterns in [`docs/07-scheduled-tasks.md`](../../docs/07-scheduled-tasks.md).

---

## What It Does

Every weekday at 7:00am:

1. Pulls the top 5 Hacker News stories
2. Checks the weather for your location
3. Reads yesterday's git commits across `~/code/*`
4. Summarizes unread emails marked important (if Gmail MCP is connected)
5. Sends one Telegram message with the whole thing

---

## Prerequisites

- `cron` and `messaging` extras installed
- Telegram gateway running ([chapter 6](../../docs/06-messaging-gateways.md))
- Optional: Gmail MCP server connected for the email summary

---

## Setup

### Option A: Just Tell Hermes

From a Hermes session:

```
> Every weekday at 7am, send me a morning briefing on Telegram with:
  - Top 5 Hacker News stories (one-line summaries)
  - Today's weather for San Francisco
  - My git commits from yesterday in ~/code/*
  - Any important unread emails from the last 24h
  Keep it under 300 words. Use bullet points.
```

Hermes appends a new job entry to `~/.hermes/cron/jobs.json` and confirms:

```
✓ Scheduled "morning-briefing"
  Schedule: Mon-Fri at 07:00 (cron: 0 7 * * 1-5)
  Next run: tomorrow at 07:00
  Delivery: Telegram
```

### Option B: Hand-Edit `jobs.json`

All cron jobs live in a single file, `~/.hermes/cron/jobs.json`. You can append a job entry directly if you prefer not to go through the natural-language interface.

The entry for this briefing looks roughly like:

```json
{
  "name": "morning-briefing",
  "schedule": "0 7 * * 1-5",
  "prompt": "Produce my morning briefing. Keep it under 300 words total.\n\nSections (bullet points only, no preamble):\n1. HACKER NEWS — top 5 stories from news.ycombinator.com, one line each\n2. WEATHER — today's forecast for San Francisco, CA\n3. YESTERDAY'S COMMITS — git log --since=yesterday across ~/code/*\n   (one bullet per repo with commit count and topic)\n4. EMAIL — unread important emails from last 24h, subject + sender only\n   (skip section if Gmail isn't connected)\n\nDo not add a greeting or a closing. Just the sections.",
  "deliver": "telegram",
  "model": null,
  "provider": null
}
```

The recipient user ID is picked up from the `TELEGRAM_ALLOWED_USER_IDS` entry in `~/.hermes/.env` — the cron job doesn't restate it. Leave `model` and `provider` as `null` to inherit your global config, or fill them in to override for this job only (see the "Model choice" section below).

Back up `jobs.json` before editing by hand. The scheduler re-reads the file on its next tick, so no restart is needed — but invalid JSON will surface as a scheduler error, so verify the file parses (`python -m json.tool < ~/.hermes/cron/jobs.json`) before walking away.

---

## Test It Before the First Real Run

Don't wait until 7am tomorrow to find out the prompt is broken.

```bash
hermes cron run morning-briefing
```

The task fires immediately. Check Telegram. Iterate on the prompt until the output is genuinely useful, then wait for the scheduler to take over.

---

## Sample Output

```
📰 HACKER NEWS
• GPT-6 leak suggests 10M token context — 892 pts
• Rust Foundation announces new governance model — 423 pts
• Show HN: I built a terminal file manager in 200 LOC — 378 pts
• Why we moved off Kubernetes — 341 pts
• Ask HN: What are you shipping this week? — 287 pts

🌤️ WEATHER — San Francisco
High 68°F, low 54°F. Partly cloudy. Fog clearing by 10am. 0% rain.

💻 YESTERDAY'S COMMITS
• myproject (3 commits): auth refactor, dependency bumps
• side-project (1 commit): README update
• dotfiles (2 commits): new tmux config

📧 EMAIL (3 important unread)
• "Q3 planning doc — review by Friday" — boss@company.com
• "PR #128 needs your review" — github@noreply
• "Your flight to NYC is tomorrow" — delta@mail.delta.com
```

---

## Tuning the Prompt

Things worth adjusting after your first few runs:

**Length** — if 300 words is too much or too little, change it. The agent honors word budgets well.

**Section order** — put what you care about most at the top. You're more likely to read to the end of one section than all four.

**Skip empty sections** — add: *"If a section has nothing to report, omit it entirely rather than writing 'nothing to report'."*

**Timezone** — the schedule fires in the server's local time. To run at 7am Pacific on a UTC server, either set `TZ=America/Los_Angeles` in the systemd unit, or change the cron to `0 15 * * 1-5` (UTC equivalent).

**Model choice** — this task doesn't need your smartest model. Briefings are well-served by cheaper models like `gpt-4o-mini` or `claude-haiku-4-5`. Override just for this task by setting the top-level `model` and `provider` scalar fields on the job entry:

```json
{
  "name": "morning-briefing",
  "schedule": "0 7 * * 1-5",
  "prompt": "...",
  "deliver": "telegram",
  "model": "anthropic/claude-haiku-4-5",
  "provider": "openrouter"
}
```

Leave either field as `null` to inherit from your global `~/.hermes/config.yaml`.

---

## Variations

**Weekly digest** instead of daily:
```yaml
schedule: "0 9 * * 1"   # Mondays at 9am
```

**Evening wrap-up** — ask about the day you just had:
> "Every weekday at 6pm, summarize what got done today: git commits, calendar events attended, and any memories written by me during the day. Send to Telegram."

**Focus check-in** — interrupt yourself mid-afternoon:
> "Every weekday at 2pm, message me on Telegram with one question: 'What's the single most important thing to finish today?' Save my reply as a memory tagged 'focus'."

---

## Disabling / Removing

```bash
hermes cron disable morning-briefing    # pause (keeps file)
hermes cron enable morning-briefing     # resume
hermes cron delete morning-briefing     # permanent
```

Or edit `~/.hermes/cron/jobs.json` directly, remove the object with `"name": "morning-briefing"`, and save. The scheduler picks up the change on its next tick.
