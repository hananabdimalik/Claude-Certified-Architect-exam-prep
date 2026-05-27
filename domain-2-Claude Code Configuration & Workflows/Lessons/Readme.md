  # Claude Code Custom Commands — Exercises

> **Context:** These exercises are designed for the `bookmark-manager` FastAPI project. You should have already worked through `/init` and have a `CLAUDE.md` file at the project root with hard rules defined.
>
> **Goal:** Build three custom slash commands for Claude Code, each one teaching a different concept. By the end you'll understand the *why* behind frontmatter, `allowed-tools`, arguments, refusal clauses, and the read/write boundary.
>
> **Where solutions live:** Each exercise has a reference solution in the `solutions/` folder. Don't peek until you've made a genuine attempt — the learning is in the wrestling, not the reading.

---

## Before you start

Make sure you have:

- The `bookmark-manager` project open in your Codespace (or local clone)
- Claude Code installed and authenticated (`claude` runs at the terminal)
- A `CLAUDE.md` file at the project root with at least 2-3 hard rules defined
- A `.claude/commands/` folder created (create it if it doesn't exist — `mkdir -p .claude/commands`)

### Quick reference — anatomy of a custom command

A custom command is a markdown file at `.claude/commands/<name>.md`. The filename minus `.md` becomes the slash command. So `explain.md` → `/explain`.

The file has two parts:

1. **YAML frontmatter** (between `---` fences) — metadata: description, argument hint, allowed tools
2. **Body** — natural-language instructions to Claude about what the command does

Minimal example:

```markdown
---
description: A short description shown in the slash menu
argument-hint: [optional hint about what arguments look like]
allowed-tools:
  - Read
  - Glob
---
Instructions to Claude go here. You can reference $ARGUMENTS to insert
whatever the user typed after the command name.
```

After creating or editing a command file, **restart Claude Code** (`/exit` then `claude`) so it picks up the change. Type `/` at the prompt to see your command in the list.

---
