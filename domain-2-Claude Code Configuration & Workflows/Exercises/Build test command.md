## Exercise 2 — Build `/test` (write + verify)

**Difficulty:** ⭐⭐ Medium
**Concepts:** arguments, multi-step instructions, restricted Bash, refusal on missing args

### The brief

You want a command that, given a function name (e.g. `create_bookmark`), writes pytest tests for it and runs them to confirm they pass. This is a **write** command with **verification** — it modifies test files and runs the test suite.

### Acceptance criteria

Your `/test` command must:

1. Live at `.claude/commands/test.md`
2. Accept a function name as an argument (e.g. `/test create_bookmark`)
3. **Refuse to proceed if no argument is given** — do NOT guess which function to test
4. Find the function in the codebase, write tests covering happy path + at least one edge case + one error case
5. Place tests in the project's existing test structure (don't invent a new folder)
6. Run pytest to verify the tests pass — but **only** `pytest -q`, nothing else
7. If tests fail, report the failure rather than "fixing" the function under test

### Step-by-step walkthrough

**Step 1 — Frontmatter setup**

Start with the same structure as Exercise 1, but think about which tools you need:

- `Read`, `Glob`, `Grep` — to find the function and understand its signature
- `Write` — to create new test files
- `Edit` — to add tests to existing test files
- `Bash` — to run pytest

Add all five. We'll restrict `Bash` in the body of the command, not in the frontmatter.

**Step 2 — The required-argument trap**

Here's where most students slip. By default, if you say "test the function specified by `$ARGUMENTS`" and the user types just `/test` with nothing after it, Claude will try to *be helpful* and pick a function to test. This is bad — silent guessing is exactly the failure mode the exam tests you on.

Your command body must include an explicit refusal:

> "If `$ARGUMENTS` is empty or contains only whitespace, stop immediately. Tell the user: `This command requires a function name as an argument. Example: /test create_bookmark`. Do not proceed. Do not guess which function to test."

Without this, the command quietly does the wrong thing. With it, the command fails loudly and correctly.

**Step 3 — Locking down Bash**

Bash is the most dangerous tool. Once you've granted it, Claude can in principle run any shell command. You restrict it in the command body with specific language:

> "Use `Bash` to run only this exact command: `pytest -q`. Do not run any other shell command under any circumstances. Do not chain commands with `&&`, `;`, or `|`. Do not invoke `bash`, `sh`, `python`, `pip`, `git`, or anything other than `pytest -q`. If pytest is not the right verification step for some reason, stop and tell the user — do not improvise."

Three things to notice:
- **Specific allowed invocation** — the exact string, not "run tests"
- **Enumerated denials** — listing the anti-patterns names them so Claude can avoid them
- **Escape hatch** — telling Claude what to do when the rule doesn't fit, instead of leaving it to improvise

**Step 4 — Output discipline**

Tell Claude what to do when tests fail:

> "If pytest fails, report the failure output to the user. Do NOT modify the function under test to make the tests pass. The test command writes tests; it does not modify production code. If a test is wrong, fix the test. If the function appears buggy, report the bug rather than silently fixing it."

This preserves the test/code boundary. A test command that "fixes" the function under test is worse than useless — it destroys your ability to catch real bugs.

**Step 5 — Test the refusal**

After saving and restarting, the first thing you test is **the failure path**:

```
/test
```

It should refuse, not guess. If it guesses, your refusal clause isn't explicit enough.

Then test the happy path:

```
/test create_bookmark
```

It should find the function, write tests in the right place, run pytest, and report the result.

### How to know you've succeeded

- `/test` with no argument → refuses with a clear error message
- `/test create_bookmark` → finds the function, writes tests, runs pytest, reports
- `/test create_bookmark and also run git status` → runs only pytest (the Bash lockdown holds)
- If you deliberately break the function being tested, Claude reports the failure rather than "fixing" the function

### Reference solution

See `solutions/test.md`.

---
