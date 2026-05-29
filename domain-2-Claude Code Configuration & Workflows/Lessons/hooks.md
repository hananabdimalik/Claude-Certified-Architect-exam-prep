# Lesson — Pre-tool and Post-tool Hooks

> **Audience:** Students who have completed the custom commands, CLAUDE.md, and CI/CD lessons, working in the `bookmark-manager` project.
>
> **Duration:** ~50 minutes teaching + ~40 minutes hands-on.
>
> **What students walk away with:** A working `PreToolUse` hook that blocks edits to protected files, a `PostToolUse` hook that auto-formats Python, and a correct mental model of how hooks differ from tools.
>
> **A note on accuracy:** Hook configuration changes between Claude Code versions. The mechanics here reflect the official docs as of early 2026 (code.claude.com/docs/en/hooks-guide). Before teaching, skim that page for any changes, and have students run `/hooks` to see what their installed version actually supports.

---

## Part 1 — The mental model (get this right first)

### The common misconception

Many summaries say "hooks are passive observers, tools are active actors." **This is wrong, and the exam may catch students who believe it.**

A `PreToolUse` hook can block a tool call before it runs. That is not passive — it's an active gatekeeper. So the passive/active framing breaks down immediately.

### The accurate distinction

The real difference is **who decides, and whether it's guaranteed**:

- **Tools** are capabilities Claude *chooses* to use as part of its reasoning — Read, Edit, Write, Bash, and so on. Whether a tool gets used depends on the model's judgement in the moment. It's probabilistic.

- **Hooks** are deterministic shell commands that fire *automatically* at fixed points in Claude Code's lifecycle, regardless of what the model decides. They are guaranteed to run when their event fires and their matcher matches.

So the contrast is **deterministic (hooks) vs model-decided (tools)** — not passive vs active.

### Why this matters

The whole point of a hook is that it doesn't rely on the model choosing to do the right thing. If you tell Claude in CLAUDE.md "always run the formatter after editing," it *usually* will — but not always, because that's a model decision. If you put formatting in a `PostToolUse` hook, it *always* runs, because that's deterministic.

Phrase it for the class like this:

> CLAUDE.md asks Claude nicely. A hook doesn't ask — it just happens.

### Some hooks observe, some act

It's true that *some* hooks are passive observers — a logging hook that writes a line to a file and exits 0 doesn't change anything. But others actively block (`PreToolUse` exit 2), modify tool inputs (newer versions), or auto-approve permissions. The defining feature of hooks isn't passivity — it's that they're deterministic and guaranteed, whether they observe or act.

### The strongest single fact for the exam

A `PreToolUse` hook that denies a tool call blocks it **even in bypass-permissions mode or with `--dangerously-skip-permissions`**. Hooks can tighten restrictions that the user cannot loosen. This is the clearest demonstration that hooks are not "just observers" — they sit above the permission system as a policy layer.

---

## Part 2 — Where hooks live: settings.json and scopes

### The configuration file

Hooks are configured as JSON inside a `settings.json` file, under a top-level `hooks` key. There are three scopes, and which file you use determines who gets the hook:

| File | Scope | Committable? |
|------|-------|--------------|
| `~/.claude/settings.json` | All your projects (personal) | No — local to your machine |
| `.claude/settings.json` | One project | Yes — commit it, whole team gets it |
| `.claude/settings.local.json` | One project | No — gitignored, personal overrides |

For a project that wants every team member to get the same safety gates, project-level `.claude/settings.json` (committed) is the right choice. For personal preferences like desktop notifications, user-level `~/.claude/settings.json` makes more sense.

### The shape of a hook config

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "path/to/script.sh"
          }
        ]
      }
    ]
  }
}
```

Walk the class through the nesting, because it trips people up:

- `"hooks"` — the top-level key
- `"PreToolUse"` — the *event* name (when to fire)
- `"matcher"` — which tools this applies to (`Edit|Write` means Edit or Write, not Bash or Read)
- inner `"hooks"` array — the actual handlers to run
- `"type": "command"` — run a shell command (other types exist; see Part 5)
- `"command"` — what to run

The double `hooks` nesting (event → matcher group → handler list) is the part students get wrong. Emphasise it.

### Viewing and reloading

Run `/hooks` inside Claude Code to browse all configured hooks grouped by event. The menu is read-only — to change a hook you edit the JSON. Edits are usually picked up automatically by a file watcher; if not, restart the session.

---

## Part 3 — The two headline events

### PreToolUse — fires before a tool call

`PreToolUse` runs *before* Claude executes a tool. This is your chance to inspect what Claude is about to do and stop it if needed.

Use it for:
- **Validation** — check the tool input before it runs
- **Blocking dangerous operations** — refuse edits to protected files, refuse dangerous bash commands
- **Policy enforcement** — enforce team rules that can't be bypassed

The key power: a `PreToolUse` hook can *block* the action. We'll see how in Part 4.

### PostToolUse — fires after a tool succeeds

`PostToolUse` runs *after* Claude completes a tool call. The action has already happened, so you can't undo it — but you can react to it.

Use it for:
- **Auto-formatting** — run a formatter on a file Claude just edited
- **Running tests** — kick off the test suite after a change
- **Logging** — record what Claude did, after the fact
- **Cleanup** — tidy up temporary state

The key limitation: PostToolUse cannot undo. The tool already ran. If you need to *prevent* something, that's PreToolUse, not PostToolUse.

### The decision table students should memorise

| You want to... | Use |
|----------------|-----|
| Prevent an action before it happens | `PreToolUse` (block with exit 2) |
| React to an action after it happens | `PostToolUse` |
| Validate input before execution | `PreToolUse` |
| Format/test/log after execution | `PostToolUse` |

"Preventive control → PreToolUse. Reactive automation → PostToolUse." That one line covers most exam questions.

---

## Part 4 — How hooks communicate: stdin, exit codes, stderr

### The data flow

When an event fires, Claude Code sends the hook a JSON payload on **stdin**. For a `PreToolUse` hook on Bash, it looks roughly like:

```json
{
  "session_id": "abc123",
  "cwd": "/path/to/project",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

Your hook script reads this, inspects the fields it cares about (often `tool_input.command` or `tool_input.file_path`), and decides what to do.

The standard way to read it in a shell script:

```bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')
```

`jq` is the JSON-parsing tool. The `-r` flag gives raw output (no surrounding quotes).

### The exit codes — this is the heart of it

The hook tells Claude Code what to do via its **exit code**:

- **Exit 0** — no objection. The action proceeds normally. (For a `PreToolUse` hook, this does NOT auto-approve — the normal permission flow still applies. It just means "I have no opinion.")
- **Exit 2** — block the action. Write a reason to **stderr**, and Claude receives that reason as feedback so it can adjust its approach.
- **Any other exit code** — the action proceeds, but Claude Code logs a hook error.

Exit 2 is the power move. It's the only exit code that blocks. A security hook that uses exit 1 instead of exit 2 provides *no enforcement* — it just logs a warning while letting the action through. Hammer this point: **for a PreToolUse guard to actually block, it must exit 2.**

### stderr becomes Claude's feedback

When a PreToolUse hook exits 2, whatever it wrote to stderr is handed to Claude as the reason. This matters — a good block message tells Claude what to do instead:

```bash
echo "Blocked: .env files are protected. Edit a .env.example template instead." >&2
exit 2
```

Claude reads that and can adjust. A blunt "Blocked." teaches Claude nothing; a reason lets it recover.

### The policy-layer fact (exam gold)

A `PreToolUse` hook that blocks fires *before* any permission-mode check. So it blocks the tool **even if the session is running with `--dangerously-skip-permissions`**. The reverse isn't true: a hook returning "allow" can't override a deny rule in settings. Hooks can tighten, never loosen. This is the single most exam-relevant fact about hooks.

---

## Part 5 — The four handler types (brief)

Most hooks use `"type": "command"` (run a shell script). Mention the others so students recognise them:

- **`command`** — run a shell command. The default and most common.
- **`http`** — POST the event JSON to a URL and get a decision back. Useful for a shared team audit service.
- **`prompt`** — single-turn LLM evaluation. For decisions needing judgement rather than a deterministic rule (e.g. "does this commit message look reasonable?"). Uses a small model by default.
- **`agent`** — multi-turn verification with tool access. Can read files and run commands before deciding. Experimental. Example: "verify all tests pass before allowing Claude to stop."

The `prompt` and `agent` types are interesting because they reintroduce model judgement into what's otherwise a deterministic system — a hybrid. Worth a mention but not worth deep coverage for this lesson.

---

## Part 6 — The four use cases from the brief

The brief lists four things hooks are good for. Map each to a concrete pattern:

### Logging
A `PostToolUse` hook on `Bash` that appends each command to a log file:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.command' >> ~/.claude/command-log.txt" }
        ]
      }
    ]
  }
}
```
Passive observer. Records, doesn't interfere.

### Validation
A `PreToolUse` hook that checks a command for dangerous patterns and blocks with exit 2 if found. Active gatekeeper.

### Notifications
A `Notification` event hook that fires a desktop notification when Claude needs input — useful for long-running tasks where you've switched windows.

### Policy enforcement
A `PreToolUse` hook that blocks edits to protected files (`.env`, lock files, `.git/`). Because it blocks even in bypass mode, it's a genuine policy layer, not a suggestion.

Note how these split: logging and notifications are passive; validation and policy enforcement are active. Same mechanism, different exit-code behaviour. This is the concrete evidence that "hooks are passive" is too simple.

---

## Part 7 — Hands-on flow

Students do the exercise file. Suggested timing:

1. (5 min) Set up the `.claude/settings.json` structure and run `/hooks` to confirm it's empty
2. (10 min) Build a PostToolUse logging hook, trigger it, check the log
3. (15 min) Build a PreToolUse protected-files hook with a script, test that it blocks
4. (10 min) Build a PostToolUse auto-formatter
5. (10 min) Test that the protected-files block works even with `--dangerously-skip-permissions`
6. (10 min) Reflection

### What to watch for

- **Hook not firing** — usually the matcher doesn't match the tool name (case-sensitive), or the JSON has a trailing comma (invalid)
- **Block not working** — student used exit 1 instead of exit 2. Only exit 2 blocks.
- **Script not executable** — `chmod +x` was skipped. Claude Code can't run a non-executable script.
- **jq not found** — needs installing, or use Python for parsing
- **The double-nesting confusion** — students put the handler at the wrong level in the JSON

---

## Reflection questions

1. Why is "hooks are passive observers, tools are active actors" inaccurate? Give a counterexample.

2. What's the real distinction between a hook and a tool? (Hint: who decides, and is it guaranteed?)

3. Why does a security validation hook need to exit with code 2 specifically? What happens if it exits 1?

4. You want to ensure a formatter runs after every file edit. Why is a PostToolUse hook more reliable than an instruction in CLAUDE.md?

5. A PreToolUse hook blocks an edit even when the user ran Claude with `--dangerously-skip-permissions`. Why is that the case, and why is it a useful property?

6. When would you reach for a `prompt`-type hook instead of a `command`-type hook?

7. You want a hook shared with your whole team. Which settings file, and why?

---

## Appendix — how this connects to the rest of the course

Students have now seen three ways to control Claude Code's behaviour, and should understand the spectrum:

- **CLAUDE.md** — asks Claude to follow rules. Model-decided, usually followed, not guaranteed.
- **Custom commands** (`/review`, etc.) — package up instructions and tool restrictions for repeated use. Still model-driven once invoked.
- **Hooks** — deterministic enforcement at lifecycle points. Guaranteed to run; can block regardless of model judgement.

The progression is from "asking" to "enforcing." A great exam scenario ties them together: "A team's CLAUDE.md says never to edit the migrations folder, but Claude occasionally does anyway. How do you make this a hard guarantee?" The answer is a PreToolUse hook that blocks edits to that path — moving the rule from CLAUDE.md (asked) to a hook (enforced).
