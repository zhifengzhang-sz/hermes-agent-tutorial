---
name: hello-skill
description: A minimal example skill. Greets the user and demonstrates the skill frontmatter + body structure. Use when the user says "hello skill" or explicitly invokes /hello-skill.
---

# Hello Skill

This is a deliberately minimal skill to illustrate the structure. Copy it to `~/.hermes/skills/hello-skill/SKILL.md`, restart Hermes, and invoke it with `/hello-skill`.

## When to use

- The user says "run the hello skill"
- The user types `/hello-skill`
- You want to demonstrate that skills load correctly

## What to do

1. Greet the user by name if you know it from memory; otherwise a generic greeting.
2. Tell them the current time (use the `time` tool).
3. Tell them which model is active (available in your system context).
4. Stop. Do not continue into other tasks.

## What not to do

- Do not run shell commands
- Do not search the web
- Do not write to memory
- Do not produce more than 3 sentences

## Example output

> Hi Alex — it's 2:47 PM. I'm running on claude-opus-4-7 via OpenRouter. That's all this skill does.
