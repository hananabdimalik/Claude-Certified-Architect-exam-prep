## Exercise 1 — Build `/explain` (read-only)

**Difficulty:** ⭐ Easy
**Concepts:** frontmatter basics, `allowed-tools`, the read/write boundary

### The brief

You want a command that, given a Python file path, explains what the code does in plain English — what each function is for, what the overall module's responsibility is, and how it fits into the bookmark-manager architecture. This is a **read-only** command: it must never modify any file.

### Acceptance criteria

Your `/explain` command must:

1. Live at `.claude/commands/explain.md`
2. Accept a file path as an argument (e.g. `/explain app/services/bookmark_service.py`)
3. If no argument is given, ask the user which file to explain rather than guessing
4. Only have access to read-type tools — **no Write, no Edit, no Bash**
5. Produce output structured as: (a) one-sentence module summary, (b) function-by-function breakdown, (c) how the file fits the project's layered architecture

### Step-by-step walkthrough

**Step 1 — Choose the description**

Open a new file at `.claude/commands/explain.md`. The first thing in your frontmatter is the `description`. This is what shows up in the slash command menu, so it has to be short and obvious. Aim for under 60 characters.

❓ *What would you write?* Something like "Explain what a Python file does in plain English" works. Avoid vague descriptions like "Helps with code" — when there are 10 commands in the menu, the user has to pick the right one at a glance.

**Step 2 — Add an argument hint**

The `argument-hint` field tells Claude Code what to display next to the command name in the menu. Since `/explain` takes a file path, your hint should make that obvious. The convention is square brackets for arguments.

❓ *What would you write?* `[file_path]` is the standard. Some people prefer `<file_path>` (angle brackets imply required, square brackets imply optional — but Claude Code doesn't enforce a convention, so pick one and stick to it across all your commands).

**Step 3 — Decide on `allowed-tools`**

This is the critical pedagogical moment. `/explain` is read-only. Ask yourself: *what tools does Claude need to read and analyse a Python file?*

- `Read` — to read the file's contents ✅
- `Glob` — in case Claude needs to find related files (e.g. the test file for this module) ✅
- `Grep` — to search for where this function is called elsewhere ✅
- `Write` — to modify files ❌ (we don't want this)
- `Edit` — to modify files ❌ (we don't want this)
- `Bash` — to run shell commands ❌ (we don't want this — no `pytest`, no `git`, nothing)

By **omitting** the write tools from `allowed-tools`, you guarantee Claude can't accidentally "helpfully" fix something while explaining it. The read/write boundary is enforced by the tool list, not by hoping Claude behaves.

**Step 4 — Write the instructions (body of the file)**

Below the closing `---` of the frontmatter, write the instructions. A good structure:

1. **Set the role and context** — "You are explaining Python code in this FastAPI bookmark manager project."
2. **Handle the argument** — "If `$ARGUMENTS` is provided, explain that file. If not, ask the user which file to explain before proceeding."
3. **Define the output structure** — Tell Claude exactly what sections to produce. Don't leave format to interpretation.
4. **Set the audience** — "Write for a developer who is new to this codebase but comfortable with Python."

**Step 5 — Save, restart, test**

Save the file. In Claude Code, type `/exit`, then run `claude` again. Type `/` — you should see `/explain` in the list. Test it with:

```
/explain app/services/bookmark_service.py
```

Then test it without an argument:

```
/explain
```

It should ask you which file rather than guessing.

### How to know you've succeeded

- The command appears in the slash menu with your description
- Running it on a file produces a structured explanation
- Running it without an argument prompts you for one
- Asking Claude to "also fix the bugs while you're at it" — Claude refuses or says it doesn't have the tools (because Write/Edit aren't in `allowed-tools`)

That last test is the most important one. If Claude *can* modify the file when asked, your `allowed-tools` is wrong.

### Reference solution

See `solutions/explain.md`.

---
