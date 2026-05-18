# Domain 2 — Claude Code Configuration & Workflows

**Exam Weight: 20%**

This domain tests your ability to configure Claude Code as a production-grade development tool. It covers CLAUDE.md, settings, hooks, permissions, custom commands, and CI/CD integration. Expect highly specific questions about configuration hierarchy and the `-p` flag.

---

## Topics to Cover

### 3.1 CLAUDE.md — The Project Brain
- What CLAUDE.md is and why it matters: it acts as the persistent "tech lead" for every Claude Code session
- CLAUDE.md hierarchy: global (`~/.claude/CLAUDE.md`) → project root → subdirectory
- What to put in CLAUDE.md: architecture rules, naming conventions, tech stack, patterns to follow/avoid, current work context
- How Claude Code loads and merges multiple CLAUDE.md files
- Writing a CLAUDE.md that actually constrains Claude's behaviour vs. one that gets ignored

### 3.2 Settings & Permissions
- `.claude/settings.json` structure: `permissions`, `mcpServers`, `hooks`, `env`
- Permission levels: what Claude Code can and cannot do by default
- `allowedTools` and `disallowedTools` — restricting Claude in sensitive contexts
- User-level settings (`~/.claude/settings.json`) vs. project-level (`.claude/settings.json`)
- Why a new team member not receiving instructions is almost always a scope mismatch

### 3.3 Custom Commands
- Creating custom slash commands in `.claude/commands/`
- Command frontmatter: `description`, `allowed-tools`, `context`, `argument-hint`
- `context: fork` — running a command in an isolated subagent so verbose output doesn't pollute the main session
- Unifying commands and skills: both live in `.claude/commands/` (or `.claude/skills/`) and create `/name` shortcuts
- Use cases: `/review`, `/test`, `/deploy`, `/document`

### 3.4 Hooks
- Pre-tool and post-tool hooks — intercepting Claude Code actions
- Using hooks for: logging, validation, sending notifications, enforcing policies
- Hook configuration in `settings.json`
- Difference between hooks and tools — hooks are passive observers, tools are active actors

### 3.5 Non-Interactive Mode & CI/CD
- The `-p` flag (print mode): runs Claude Code non-interactively and exits — **without it, CI jobs hang indefinitely**
- `--output-format json` for machine-readable output in pipelines
- `--max-turns` for controlling how many agentic turns a CI job can take
- `--allowedTools` for restricting Claude's permissions in automated contexts
- Safe patterns for using Claude Code in GitHub Actions

### 3.6 Claude Code in GitHub Actions
- The `anthropics/claude-code-action` — triggering Claude from issue/PR comments with `@claude`
- Storing `ANTHROPIC_API_KEY` as a GitHub Actions secret
- Using Claude Code to: auto-review PRs, implement issues, run code modernisation tasks
- Security considerations: what Claude Code can and cannot access in a runner

---

## Hands-On Projects

### ⚙️ Project 1 — Production-Ready CLAUDE.md + Command Suite

Set up a realistic project configuration that a team could actually use:

- Write a `CLAUDE.md` at the project root that defines: stack, folder structure, naming conventions, testing approach, and 3 explicit "never do this" rules
- Create at least 4 custom commands:
  - `/review` — reviews the current file for bugs and style issues
  - `/test` — generates unit tests for the current function
  - `/explain` — produces a plain-English explanation of a selected code block
  - `/refactor` — refactors the current function with a reason argument
- One command must use `context: fork` to run in isolation
- One command must use `allowed-tools` to restrict what Claude can access
- Add a post-tool hook that logs every file write to `audit.log`

**What you'll learn:** CLAUDE.md hierarchy, command frontmatter, hooks, permission scoping

**Stretch goal:** Create a subdirectory with its own `CLAUDE.md` that overrides one rule from the root — then verify Claude follows the more specific rule inside that folder.

---

### 🔄 Project 2 — Claude Code in a CI/CD Pipeline

Integrate Claude Code into a GitHub Actions workflow:

- Set up a workflow that triggers on every pull request
- Use the `-p` flag to run Claude Code non-interactively: review the diff and output a JSON summary of issues found
- Parse the JSON output and post it as a PR comment using the GitHub API
- Add a second workflow triggered by `@claude` mentions in issue comments that can implement small fixes
- Store the API key correctly as a GitHub Actions secret
- Add `--max-turns 5` to prevent runaway agentic loops in CI

**What you'll learn:** `-p` flag (critical exam topic), JSON output parsing, GitHub Actions integration, security practices

**Stretch goal:** Add a conditional step that fails the CI check if Claude Code finds any issues rated `critical` in its JSON output, turning it into a quality gate.

---

## Folder Structure (for contributors)

```
domain-2-claude-code-configuration/
├── README.md           ← this file
├── version-a/          ← Contributor A's course
│   ├── lessons/
│   ├── exercises/
│   └── solutions/
└── version-b/          ← Contributor B's course
    ├── lessons/
    ├── exercises/
    └── solutions/
```

## Key Anti-Patterns to Teach (exam traps)

- **Not using `-p` in CI** — the single most common CI integration mistake, causes pipelines to hang indefinitely
- Putting project-specific config in user-level settings — affects every project on the machine
- Writing a CLAUDE.md that's too vague to constrain anything ("write clean code")
- Forgetting `context: fork` on heavy commands — they pollute the main session context
- Not restricting `allowedTools` in automated/CI contexts — Claude could make unintended changes

