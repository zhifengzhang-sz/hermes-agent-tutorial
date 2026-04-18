# Contributing

Thanks for wanting to improve this tutorial. It's a living doc — Hermes Agent moves fast, and keeping the instructions accurate is the main maintenance burden.

## What's Welcome

- **Typo and grammar fixes** — just open a PR
- **Corrections when the official docs change** — link to the updated source
- **Clarifications** — if a step confused you, it'll confuse the next person
- **New exercises** — especially for skills you built that were useful
- **Additional examples** — worked walkthroughs of real workflows
- **Translations** — into any language

## What's Not Welcome

- **Duplicating the official docs** — link to them instead
- **Opinions framed as facts** — "this model is the best" without benchmarks
- **Screenshots of your terminal** — keep it text-only so copy/paste works
- **Promotional content** — no affiliate links, no "check out my course"

## How to Contribute

1. Fork the repo
2. Create a branch named after the change (`fix/installation-termux-path`, `add/discord-gateway-example`)
3. Make your changes
4. Test any commands or snippets you added or modified on a real install
5. Open a PR with a short description of what changed and why

## Style Guide

A few house conventions:

- **Prose over bullet lists** where the content is explanation, not a checklist
- **Second-person** ("you'll see..." not "users will see...")
- **Present tense** for describing what Hermes does ("Hermes writes a file" not "Hermes will write")
- **Specific beats generic** — "run `hermes doctor`" beats "check your installation"
- **Short commands in `code spans`; multi-line in fenced blocks** with language tags
- **Link to official sources** at the top of each chapter and inline when quoting behavior
- **No marketing voice** — this is a tutorial, not a launch post

## Verifying Commands

Before submitting a PR that adds or changes commands:

1. Install Hermes fresh (VM, container, or `rm -rf ~/.hermes && re-install`)
2. Run every command in your diff
3. Confirm output matches what you described

This matters more than almost anything else. A tutorial with broken commands is worse than no tutorial.

## Questions

Open a [GitHub Discussion](https://github.com/zhifengzhang-sz/hermes-agent-tutorial/discussions) or ping in the [Nous Research Discord](https://discord.gg/NousResearch).

---

## Code of Conduct

Be decent. Assume good faith. If someone's PR is wrong, say why and link evidence. If someone's PR is right and yours was wrong, update yours. That's the whole thing.
