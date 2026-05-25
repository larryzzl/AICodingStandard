# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

<!-- reason: Clarifies that this file governs agent behavior while keeping the expected rule-file naming discoverable for agents. -->
This file controls agent behavior. Stack-specific rule files, commonly named `RULES.md`, control implementation constraints. Both must be followed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 0. Instruction Priority

<!-- reason: Agents need a deterministic priority order when user requests, architecture rules, and local style appear to conflict. -->
Follow instructions in this order:

1. User's current request.
2. Project-specific rule files, commonly named `RULES.md`.
3. This behavioral guideline.
4. Existing code style and conventions.

<!-- reason: Conflict handling should happen before code edits, not after an incorrect implementation. -->
If instructions conflict, stop and explain the conflict before editing code.

## 1. Repository Discovery

<!-- reason: Naming the expected rule file makes discovery reliable while still allowing each stack to define its own constraints. -->
Before coding in this project:
- Read the relevant stack-specific rule file, commonly named `RULES.md`, when the task touches architecture, data flow, integration boundaries, security, platform behavior, or tests.
- Prefer the nearest applicable rule file to the files being changed.
- Inspect nearby existing code before choosing patterns.
- Prefer existing project conventions over generic best practices.

## 2. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
<!-- reason: This keeps clarification behavior useful without causing unnecessary user interruptions. -->
- If uncertainty can be resolved by reading code, configs, tests, or docs in the repo, inspect them first.
- Ask the user only when the requirement has multiple product-level meanings, the change could affect public API, data model, security, or architecture, or proceeding would require guessing business behavior.

## 3. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 4. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

<!-- reason: Diff discipline prevents agents from mixing requested work with unrelated cleanup or formatting churn. -->
Do not:
- Reformat entire files unless formatting is the task.
- Rename public symbols unless necessary.
- Change generated files manually.
- Modify lockfiles unless dependencies changed.
- Mix unrelated cleanup with feature or bug-fix work.

## 5. Rule Violation Handling

<!-- reason: Project-specific rule violations should be surfaced before implementation, not hidden inside generated code. -->
If the requested implementation appears to violate a project-specific rule file:
- Do not implement the violating approach silently.
- Explain the violated rule.
- Propose the smallest compliant alternative.
- Continue only with a compliant implementation or explicit user direction.

## 6. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

<!-- reason: Verification expectations make the agent prove the change and report limits when proof is unavailable. -->
After changes:
- Run the narrowest relevant tests first.
- If tests cannot be run, explain why and state what was verified manually.
- Summarize changed files and behavior, not implementation trivia.
- Mention any remaining risk or follow-up only if it matters.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
