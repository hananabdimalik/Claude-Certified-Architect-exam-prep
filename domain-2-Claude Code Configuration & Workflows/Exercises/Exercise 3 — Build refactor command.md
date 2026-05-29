
## Exercise 3 — Build `/refactor` (risky write)

**Difficulty:** ⭐⭐⭐ Hard
**Concepts:** hard rules, refusal clauses, signature preservation, cross-file boundaries

### The brief

You want a command that refactors a function for readability — extracting helpers, renaming local variables, simplifying logic — **without changing its behaviour**. This is the riskiest command yet: it modifies production code. Every safeguard matters.

### Acceptance criteria

Your `/refactor` command must:

1. Live at `.claude/commands/refactor.md`
2. Accept a function name as an argument
3. Refuse if no argument is given
4. **Preserve the function's signature** (name, parameters, return type) unless the user explicitly says otherwise
5. **Refuse to violate any rule in CLAUDE.md** — and explain which rule when refusing
6. **Refuse to touch other files** — if the refactor would require changes to call sites or imports elsewhere, refuse and tell the user it's out of scope
7. **Refuse if the function doesn't exist** — don't invent one
8. After refactoring, run `pytest -q` to verify behaviour is preserved

### Step-by-step walkthrough

**Step 1 — Recognise this is the dangerous one**

Read the acceptance criteria again. Notice how many of them are *refusals* — four out of eight. That's the lesson of this exercise: **risky write commands are mostly defined by what they refuse to do**.

A command without refusals is a command that does whatever Claude thinks is reasonable. That's fine for `/explain`. It's catastrophic for `/refactor`.

**Step 2 — Frontmatter**

Same tools as `/test`: Read, Glob, Grep, Write, Edit, Bash. The dangerous capability isn't in the tool list — it's in what you let those tools do.

**Step 3 — The four refusal clauses**

In the body, write a `## Refusal clauses` section listing exactly four scenarios:

(a) **CLAUDE.md hard-rule violations.** If the refactor would violate any rule in CLAUDE.md, refuse and quote the rule. Example: if a hard rule says "service functions never raise HTTPException directly," and the user asks to refactor a service function to raise HTTPException, refuse.

(b) **Signature changes.** If the refactor would change the function name, parameters, parameter types, or return type, refuse. The exception is if the user explicitly says "change the signature to X" — then it's an opt-in, not an accident.

(c) **Cross-file changes.** If the refactor would require updating call sites in other files, refuse. Tell the user: "This refactor touches more than one file. Please break it into smaller refactors, or use a different command designed for multi-file changes."

(d) **Function not found.** If the named function doesn't exist anywhere in the codebase, refuse. Don't invent it.

Each refusal must **explain why** — refusal without reasoning is just frustration.

**Step 4 — Verification step**

After the refactor, run `pytest -q` with the same Bash lockdown as Exercise 2. If tests fail, **do not modify the tests** to make them pass — that defeats the entire point of verification. Report the failure.

**Step 5 — Test every refusal**

This exercise's success criteria are the refusals. Test them:

- `/refactor` with no arg → refuses
- `/refactor a_function_that_doesnt_exist` → refuses
- `/refactor create_bookmark and change it to return a dict instead of a model` → refuses (signature change)
- `/refactor create_bookmark and update all the call sites` → refuses (cross-file)
- `/refactor create_bookmark to raise HTTPException directly` (assuming CLAUDE.md forbids this) → refuses and quotes the rule

Only after all five refusals work should you test the happy path:

- `/refactor create_bookmark` → refactors the function for readability, preserves the signature, runs pytest, reports results

### How to know you've succeeded

- Every refusal triggers correctly with a clear explanation
- The happy-path refactor preserves the signature exactly
- pytest still passes after the refactor (behaviour preserved)
- No other files were modified

### Reference solution

See `solutions/refactor.md`.

---

## Reflection questions

After finishing all three exercises, write short answers to these in your notes. These are the kinds of questions the exam will ask in scenario form.

1. Why is the read/write boundary enforced through `allowed-tools` rather than instructions in the body? What's the difference in guarantee?

2. Why do risky write commands need *explicit* refusal clauses rather than relying on Claude's general judgement?

3. What's the difference between restricting `Bash` in `allowed-tools` (presence/absence) versus restricting it in the command body (allowed commands)? When would you use each?

4. Why is "do not modify the function under test to make tests pass" an essential instruction in `/test`? What failure mode does it prevent?

5. If a `/refactor` command had no cross-file refusal, what's the worst thing that could happen?

---

## Submission

When you're done:

1. Push your `.claude/commands/` folder to your fork of the repo
2. Compare your commands to the reference solutions in `solutions/`
3. Note any differences and decide whether your approach or the solution's approach is better — there isn't always one right answer
