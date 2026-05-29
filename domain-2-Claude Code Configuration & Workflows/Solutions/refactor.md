---
description: Refactor a function for readability while preserving behaviour
argument-hint: [function_name]
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Edit
  - Bash
---
You are refactoring a function in this FastAPI bookmark manager project for readability. The function's behaviour must be preserved exactly.

## Required argument

If `$ARGUMENTS` is empty or contains only whitespace, stop immediately and tell the user:

> This command requires a function name as an argument. Example: `/refactor create_bookmark`

Do not proceed. Do not guess. Refuse and exit.

## Refusal clauses

Refuse the request — with a clear explanation pointing to the specific clause that applies — in any of these situations:

### 1. CLAUDE.md hard-rule violations

Before refactoring, read `CLAUDE.md` at the project root and check whether the refactor would violate any hard rule.

If the user's request would require violating a rule (e.g. "refactor `create_bookmark` to raise HTTPException directly" when CLAUDE.md says service functions never raise HTTPException), refuse and quote the specific rule.

### 2. Signature changes

Preserve the function's signature exactly: name, parameter names, parameter types, return type, decorators.

If the refactor would change any of these, refuse — unless the user has explicitly stated they want a signature change (e.g. "rename it to `create_new_bookmark`" or "change the return type to a dict"). Implicit signature changes that emerge from the refactor are not allowed.

If you find yourself wanting to change the signature mid-refactor, stop and tell the user what you want to change and why. Let them decide.

### 3. Cross-file changes

This command refactors **one function in one file**. If the refactor would require any of:

- Updating call sites in other files
- Changing imports in other files
- Modifying tests to match new internal structure
- Changing the function's containing class or module

…then refuse. Tell the user:

> This refactor would require changes to more than one file. Please break it into smaller refactors, or use a multi-file refactoring approach outside this command.

### 4. Function not found

Use Grep to locate the function. If it doesn't exist in the codebase, refuse. Do not invent it. Tell the user the function couldn't be found and suggest they double-check the name.

If the function name is ambiguous (matches multiple definitions), ask the user which one they mean before proceeding.

## Workflow (only if no refusal clause triggers)

1. **Read the function** and any directly related code needed to understand it (the model it returns, helpers it calls in the same file).

2. **Read the existing tests** for this function, if any. You will need them to verify behaviour after the refactor.

3. **Plan the refactor.** Identify what makes the function hard to read: long nested conditionals, repeated patterns, unclear variable names, missing helper extractions. State your plan briefly before making changes.

4. **Apply the refactor** in the same file only. Acceptable changes:
   - Extract local helper functions (in the same file)
   - Rename local variables for clarity
   - Simplify conditionals (early returns, guard clauses)
   - Reduce nesting
   - Improve docstrings and comments
   
   Unacceptable changes (these trigger refusal clauses):
   - Changing the function's signature
   - Moving code to a different file
   - Changing what the function does

5. **Verify** by running pytest.

## Verification — Bash lock-down

Use `Bash` to run only this exact command:

```
pytest -q
```

Do not run any other shell command under any circumstances. Do not chain commands with `&&`, `;`, or `|`. Do not invoke `bash`, `sh`, `python`, `pip`, `git`, or anything other than `pytest -q`.

## When tests fail after refactor

If pytest fails after the refactor, the refactor has changed behaviour — which violates the contract of this command.

**Do NOT modify the tests to make them pass.** The tests are the verification that behaviour is preserved. Modifying them defeats the entire purpose.

Instead:

1. Report the failure to the user
2. Offer to revert the refactor
3. If the user wants to investigate, share what changed and ask them how to proceed

## Output

After completing the workflow, tell the user:

- A short summary of what you changed (e.g. "extracted validation into `_validate_url` helper, replaced nested ifs with early returns")
- The pytest result
- Any observations about the function that you didn't act on (e.g. "there's a potential bug in error handling, but fixing it would change behaviour — flagging for review")
