---
description: Explain what a Python file does in plain English
argument-hint: [file_path]
allowed-tools:
  - Read
  - Glob
  - Grep
---
You are explaining Python code in this FastAPI bookmark manager project to a developer who is new to the codebase but comfortable with Python.

## File to explain

If `$ARGUMENTS` is provided, explain that file.

If `$ARGUMENTS` is empty, ask the user which file they want explained before proceeding. Do not guess.

## What to produce

Structure your response in three sections, in this order:

1. **Module summary** — One sentence describing what this file is responsible for in the project.

2. **Function-by-function breakdown** — For each function or class in the file:
   - Its name and signature
   - What it does (one or two sentences)
   - Any non-obvious behaviour, side effects, or dependencies

3. **Architectural fit** — Two or three sentences explaining how this file fits into the project's layered architecture (routes → services → repositories → models). Reference the actual layers used in this project, not generic descriptions.

## Constraints

- This is a read-only command. You do not have Write, Edit, or Bash tools — do not offer to "fix" or "improve" the code, even if the user asks. If the user requests changes, tell them to use a different command (`/refactor` for changes, `/test` for adding tests).

- If the file contains code that appears buggy or violates a rule in `CLAUDE.md`, note it in your explanation as an observation — but do not modify anything.

- Use Grep and Glob if you need context from other files (e.g. to understand how a function is called elsewhere). Don't speculate about usage when you can check.

- Write for a developer new to the codebase. Avoid jargon that requires familiarity with the rest of the project. When you must use project-specific terms, define them on first use.
