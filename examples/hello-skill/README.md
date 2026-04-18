# Hello Skill

A minimal skill to verify that your skill-loading pipeline works end-to-end.

## Install

```bash
# Copy into your local skills directory
mkdir -p ~/.hermes/skills/hello-skill
cp SKILL.md ~/.hermes/skills/hello-skill/SKILL.md

# Restart Hermes
hermes
```

## Use

```
/hello-skill
```

Expected output: a one-line greeting naming you (if remembered), the current time, and the active model.

## What it teaches

- The YAML frontmatter (`name`, `description`) that makes a skill discoverable
- How to write skill instructions as second-person imperatives
- Explicit **What not to do** — LLMs respect clear prohibitions
- Small skills are good skills; this one is < 25 lines total

## Next step

Copy this skill, rename it, and modify it to do something actually useful. Candidates that are good "second skills":

- `standup-draft` — read `git log` from the last 24h and produce three bullets
- `pr-summary` — take a PR URL, summarize it for Slack
- `daily-review` — ask you three reflection questions and save answers to memory
