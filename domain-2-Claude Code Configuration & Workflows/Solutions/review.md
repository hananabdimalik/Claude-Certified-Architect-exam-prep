---
description: Review the current file for bugs, style issues, and rule violations
argument-hint: [file_path]
allowed-tools:
  - Read
  - Glob
  - Grep
---
You are reviewing Python code in this FastAPI bookmark manager project.

## File to review

If `$ARGUMENTS` is provided, review that file.

Otherwise, ask the user which file they want reviewed before proceeding. Do not guess.

## What to check

Review the file against these criteria, in this order of importance:

1. **CLAUDE.md hard rules.** Read `CLAUDE.md` at the project root and check whether the file violates any rule defined there. Hard-rule violations are the highest-priority issue — flag them first, quote the specific rule, and point to the line(s) that violate it.

2. **Architectural fit.** Does the file's content match its layer in the project?
   - Routes should be thin — parsing requests, calling a service, returning a response. Business logic in a route is an issue.
   - Services should contain business logic and orchestrate repositories. Direct database access in a service is an issue.
   - Repositories should contain data access only. Business logic in a repository is an issue.
   - Models should be data shapes. Behaviour in a model is an issue.

3. **Bugs and correctness.** Look for:
   - Unhandled error cases
   - Off-by-one errors, incorrect conditionals
   - Race conditions or ordering issues
   - Incorrect exception handling (catching too broad, swallowing errors silently)
   - Type mismatches between annotation and actual behaviour

4. **Style and clarity.** Lower priority, but worth noting:
   - Functions that are too long or do too many things
   - Unclear variable names
   - Missing or misleading docstrings
   - Inconsistent formatting compared to the rest of the codebase

## What NOT to do

This is a **read-only** command. You have Read, Glob, and Grep — no Write, Edit, or Bash.

- Do not modify the file.
- Do not "fix" issues as you find them, even if the fix seems obvious.
- Do not run any commands.
- If the user asks you to fix something during the review, tell them to use `/refactor` (for behaviour-preserving changes) or to make the change themselves. The review/edit boundary is what makes this command useful — if reviews silently become edits, the user loses the ability to evaluate changes before they happen.

## Output format

Structure your review as:

1. **Summary** — One sentence: is the file in good shape, or are there serious concerns?

2. **Hard-rule violations** — List each violation with the rule quoted and the line(s) involved. If none, say so explicitly.

3. **Architectural issues** — List any layer-boundary violations. If none, say so explicitly.

4. **Bugs and correctness concerns** — List with line numbers and a one-sentence explanation each.

5. **Style observations** — Lower-priority notes, listed but not belaboured.

6. **Overall recommendation** — One line: ready as-is, needs minor cleanup, needs significant rework, or has blocking issues.

Use Grep and Glob freely to check how this file is used elsewhere — reviewing a service function is much easier when you've seen which routes call it.
