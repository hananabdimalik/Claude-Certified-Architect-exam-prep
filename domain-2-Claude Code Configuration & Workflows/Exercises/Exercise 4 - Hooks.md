# Exercise 4 — Pre-tool and Post-tool Hooks

> **Context:** For the `bookmark-manager` project. You should have Claude Code installed and a working `.claude/` folder from earlier exercises.
>
> **Goal:** Build three hooks — a logging hook, a protected-files guard, and an auto-formatter — and prove to yourself that a PreToolUse guard blocks actions even when permissions are bypassed.
>
> **Where the solution lives:** Reference configs and scripts are in `solutions/hooks/`. Try first.
>
> **Version note:** Run `/hooks` early to confirm what your installed Claude Code version supports. Hook features change between versions; if something here behaves differently, check `/hooks` and the docs at code.claude.com/docs/en/hooks-guide.

---

## Before you start

You need:
- The `bookmark-manager` project open with Claude Code installed
- `jq` installed (`brew install jq`, `apt-get install jq`, or equivalent). Check with `jq --version`.
- A `.claude/` folder in the project (create with `mkdir -p .claude/hooks` if needed)

### Quick reference — the mental model

A **hook** is a deterministic shell command that fires automatically at a lifecycle point. Unlike a tool (which Claude *chooses* to use), a hook is *guaranteed* to run when its event fires. Some hooks just observe (logging); others actively block (a PreToolUse guard exiting 2).

The two events you'll use:
- **PreToolUse** — fires *before* a tool runs. Can block it (exit 2).
- **PostToolUse** — fires *after* a tool succeeds. Cannot undo; used for formatting, testing, logging.

---

## Exercise 1 — A logging hook (passive observer)

**Difficulty:** ⭐ Easy
**Concepts:** the settings.json structure, matchers, reading stdin JSON, PostToolUse

### The brief

Build a `PostToolUse` hook that logs every Bash command Claude runs to a file. This is the simplest kind of hook — it observes and records, never interferes.

### Acceptance criteria

1. A `.claude/settings.json` with a `PostToolUse` hook matching `Bash`
2. Each Bash command Claude runs gets appended to a log file
3. The hook never blocks or interferes with anything

### Step-by-step

**Step 1 — Create the settings file**

Create `.claude/settings.json` with this content:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.command' >> .claude/bash-log.txt"
          }
        ]
      }
    ]
  }
}
```

Understand the nesting before moving on:
- `"hooks"` → top-level key
- `"PostToolUse"` → the event
- `"matcher": "Bash"` → only fires for Bash tool calls
- inner `"hooks"` → the handler(s)
- `"command"` → reads the JSON on stdin, extracts the command with jq, appends to the log

**Step 2 — Reload and verify**

Restart Claude Code (`/exit` then `claude`), then run `/hooks`. You should see your PostToolUse hook listed under the Bash matcher. If it's not there, your JSON probably has a syntax error (trailing comma is the usual culprit).

**Step 3 — Trigger it**

Ask Claude to do something that uses Bash, e.g. "list the files in the app directory." After it runs, check the log:

```bash
cat .claude/bash-log.txt
```

You should see the command Claude ran.

### How to know you've succeeded

- `/hooks` shows the hook
- After Claude runs a Bash command, the command text appears in `.claude/bash-log.txt`
- Claude's behaviour is otherwise completely unaffected

### Why this is the "observer" archetype

This hook exits 0 implicitly (jq succeeds), so it never blocks. It's a pure side effect — recording, not controlling. Hold onto this contrast for Exercise 2, where the hook actively blocks.

### Reference solution

`solutions/hooks/settings-logging.json`

---

## Exercise 2 — A protected-files guard (active gatekeeper)

**Difficulty:** ⭐⭐⭐ Hard
**Concepts:** PreToolUse, blocking with exit 2, stderr feedback, script-based hooks, the policy layer

### The brief

Build a `PreToolUse` hook that blocks Claude from editing protected files — `.env`, lock files, and anything in `.git/`. When blocked, Claude should receive a clear reason so it can adjust.

### Acceptance criteria

1. A script at `.claude/hooks/protect-files.sh` that checks the target file path
2. The script exits 2 (blocks) when the path matches a protected pattern, writing a reason to stderr
3. The script exits 0 (allows) otherwise
4. A `PreToolUse` hook in settings.json matching `Edit|Write` that runs the script
5. Attempting to edit `.env` is blocked; editing a normal file is allowed

### Step-by-step

**Step 1 — Write the guard script**

Create `.claude/hooks/protect-files.sh`:

```bash
#!/bin/bash
# protect-files.sh — blocks edits to sensitive files

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

PROTECTED_PATTERNS=(".env" "package-lock.json" "poetry.lock" ".git/")

for pattern in "${PROTECTED_PATTERNS[@]}"; do
  if [[ "$FILE_PATH" == *"$pattern"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$pattern'. Edit a template or ask a human instead." >&2
    exit 2
  fi
done

exit 0
```

Understand the key lines:
- `INPUT=$(cat)` reads the JSON payload from stdin
- `jq -r '.tool_input.file_path // empty'` extracts the file path (the `// empty` gives a safe fallback if absent)
- The loop checks each protected pattern
- **`exit 2`** is what blocks — not exit 1, which would only log a warning and let the edit through
- The `>&2` sends the message to stderr, which becomes Claude's feedback

**Step 2 — Make it executable**

This is the step everyone forgets:

```bash
chmod +x .claude/hooks/protect-files.sh
```

If you skip this, Claude Code can't run the script and the hook silently fails.

**Step 3 — Register the hook**

Add a `PreToolUse` block to `.claude/settings.json`. If you still have the logging hook from Exercise 1, add this as a *sibling* event key (don't replace the whole `hooks` object):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/protect-files.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.command' >> .claude/bash-log.txt" }
        ]
      }
    ]
  }
}
```

`$CLAUDE_PROJECT_DIR` is a variable Claude Code sets to the project root, so the script is found regardless of the working directory.

**Step 4 — Test the block**

Restart Claude Code. Then test it manually first (faster than going through Claude):

```bash
echo '{"tool_name":"Edit","tool_input":{"file_path":".env"}}' | .claude/hooks/protect-files.sh
echo "Exit code: $?"
```

You should see the "Blocked" message and "Exit code: 2".

Then test a safe path:

```bash
echo '{"tool_name":"Edit","tool_input":{"file_path":"app/main.py"}}' | .claude/hooks/protect-files.sh
echo "Exit code: $?"
```

No message, "Exit code: 0".

**Step 5 — Test through Claude**

Ask Claude: "Add a comment to the top of the .env file." It should be blocked, and Claude should report that it can't edit protected files (reading the stderr you wrote).

Then ask it to edit a normal file — that should work fine.

### How to know you've succeeded

- Manual test: `.env` → exit 2 with message; normal file → exit 0 silent
- Through Claude: editing `.env` is refused with your reason; editing a normal file works
- Claude adjusts its approach based on the stderr feedback rather than just failing blindly

### The common mistake

Using `exit 1` instead of `exit 2`. Exit 1 does NOT block — the edit goes through and you just get a logged warning. Only exit 2 blocks. If your guard isn't blocking, this is almost always why.

### Reference solution

`solutions/hooks/protect-files.sh` and `solutions/hooks/settings-protect.json`

---

## Exercise 3 — An auto-formatter (reactive automation)

**Difficulty:** ⭐⭐ Medium
**Concepts:** PostToolUse for automation, why deterministic beats asking

### The brief

Build a `PostToolUse` hook that automatically formats Python files after Claude edits them, so formatting stays consistent without anyone asking Claude to do it.

### Acceptance criteria

1. A `PostToolUse` hook matching `Edit|Write`
2. After Claude edits a `.py` file, a formatter runs on it automatically
3. Non-Python files are left alone

### Step-by-step

**Step 1 — Pick a formatter**

This project is Python, so use `black` (or `ruff format`). Make sure it's installed:

```bash
pip install black --break-system-packages
```

**Step 2 — Add the hook**

Add a PostToolUse handler. Because we only want to format Python, the command checks the extension first:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(jq -r '.tool_input.file_path'); [[ \"$FILE\" == *.py ]] && black \"$FILE\" || true"
          }
        ]
      }
    ]
  }
}
```

Breaking down the command:
- `FILE=$(jq -r '.tool_input.file_path')` — get the edited file path
- `[[ "$FILE" == *.py ]] && black "$FILE"` — if it's a Python file, format it
- `|| true` — ensures the hook exits 0 even for non-Python files (so it never accidentally looks like a failure)

**Step 3 — Test it**

Ask Claude to edit a Python file in a deliberately messy way — e.g. "add a function to bookmark_service.py with bad indentation and extra blank lines." After Claude's edit, check the file — `black` should have cleaned up the formatting automatically.

### How to know you've succeeded

- After Claude edits a `.py` file, it comes out formatted, with no one asking
- Editing a non-Python file doesn't trigger the formatter
- The hook never reports an error

### The teaching point

You never told Claude to format. The hook did it deterministically. Compare this to putting "always run black after editing" in CLAUDE.md — Claude would *usually* do it, but a hook *always* does it. That reliability is the whole point of hooks.

### Reference solution

`solutions/hooks/settings-format.json`

---

## Exercise 4 — Prove the policy layer (stretch)

**Difficulty:** ⭐⭐ Medium
**Concepts:** hooks override permission modes, the strongest exam fact

### The brief

Demonstrate that your protected-files guard from Exercise 2 blocks edits *even when Claude is run with permissions bypassed*. This proves hooks are a policy layer, not just a suggestion.

### Steps

**Step 1 — Confirm the guard works normally** (you did this in Exercise 2)

**Step 2 — Launch Claude with permissions bypassed**

```bash
claude --dangerously-skip-permissions
```

Normally this flag lets Claude do anything without asking. The question: does it bypass your hook too?

**Step 3 — Try to edit a protected file**

Ask Claude to edit `.env`. 

**Expected result:** It's STILL blocked. The PreToolUse hook fires before the permission check, so even bypass mode can't get past it.

### How to know you've succeeded

- Even with `--dangerously-skip-permissions`, editing `.env` is blocked by your hook
- You understand why: hooks sit above the permission system; they can tighten what permissions allow but users can't loosen past a hook's deny

### Why this matters

This is the single most exam-relevant fact about hooks. A team can enforce "never touch these files" as a hard guarantee that no developer can override by changing their permission mode. That's genuine policy enforcement, which is why "hooks are just passive observers" is wrong.

---

## Reflection questions

1. In Exercise 1 the hook observed; in Exercise 2 it blocked. Same mechanism — what's different about how each hook uses its exit code?

2. Why must the protected-files guard exit 2 and not exit 1?

3. Why is a PostToolUse formatter more reliable than asking Claude in CLAUDE.md to format?

4. In Exercise 4, why does the guard still block under `--dangerously-skip-permissions`?

5. Could you implement the protected-files guard as a PostToolUse hook instead? Why or why not?

6. Which of your three hooks would you commit to the shared `.claude/settings.json`, and which would you keep personal in `~/.claude/settings.json`?

---

## Submission

1. Commit `.claude/settings.json` and `.claude/hooks/` to your repo
2. Confirm all three hooks appear in `/hooks`
3. Demonstrate the Exercise 4 bypass test works
4. Compare against `solutions/hooks/` and note any differences
