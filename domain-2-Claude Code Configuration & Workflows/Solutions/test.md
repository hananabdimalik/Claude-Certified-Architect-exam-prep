---
description: Write pytest tests for a function and verify they pass
argument-hint: [function_name]
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Edit
  - Bash
---
You are writing pytest tests for a function in this FastAPI bookmark manager project.

## Required argument

If `$ARGUMENTS` is empty or contains only whitespace, stop immediately and tell the user:

> This command requires a function name as an argument. Example: `/test create_bookmark`

Do not proceed. Do not guess which function to test. Do not pick "a reasonable function" to demonstrate the command. Refuse and exit.

## Workflow

If a function name was provided, proceed in this order:

1. **Locate the function.** Use Grep to find the function definition. If it appears in multiple files, ask the user which one they mean. If it doesn't exist anywhere in the codebase, stop and tell the user — do not invent it.

2. **Understand the function.** Read the file containing the function, plus any directly related files (the model it returns, the repository it calls, etc.) to understand the signature, dependencies, and side effects.

3. **Find the test location.** Use Glob to locate the existing test directory structure. Tests in this project live under `tests/` and mirror the source layout. Do not invent a new test folder. If a test file for this module already exists, add to it. If not, create a new one in the matching location.

4. **Write the tests.** Cover at minimum:
   - One happy-path test (typical valid input → expected output)
   - One edge case (boundary input, empty input, or similar)
   - One error case (invalid input → expected exception or error response)
   
   Use the existing test conventions in the project (fixtures, naming, parametrize) — read an existing test file first to match the style.

5. **Verify.** Run pytest to confirm the tests pass.

## Verification — Bash lock-down

Use `Bash` to run only this exact command:

```
pytest -q
```

Do not run any other shell command under any circumstances. Do not chain commands with `&&`, `;`, or `|`. Do not invoke `bash`, `sh`, `python`, `pip`, `git`, or anything other than `pytest -q`. If pytest is not the right verification step for some reason, stop and tell the user — do not improvise.

## When tests fail

If pytest fails, report the failure output to the user.

**Do NOT modify the function under test to make the tests pass.** This command writes tests; it does not modify production code.

- If a test you wrote is wrong (e.g. you misunderstood the function's contract), fix the test.
- If the function appears genuinely buggy, report the bug to the user as an observation. Do not fix it. The user can use a different command for that.

## Output

After completing the workflow, tell the user:

- Which file you wrote tests in
- A one-line summary of each test you added
- The pytest result (pass/fail and any failure details)
