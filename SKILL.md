---
name: apply-review
description: Apply findings from a Review-*.md file as iterative, committed fixes. Use when the user says "apply review", "fix review findings", "implement review", "apply the review", or has a Review-*.md file and wants its findings turned into code changes. Also triggers when the user references review items by ID (e.g., "fix I0", "apply I3") or says "let's work through the review."
allowed-tools: Bash, Read, Edit, Write, Glob, Grep, AskUserQuestion, Agent, TaskCreate, TaskUpdate, TaskGet, TaskList
argument-hint: "[Review-file.md]"
---

# Apply Review

## Purpose

Take a `Review-*.md` file produced by the `/review` skill and systematically apply its findings as code changes — one finding at a time, with test validation and user review before each commit.

The key value is discipline: each finding becomes an isolated, tested, committed change with a targeted commit message. The user controls staging and can reject or modify any change before it lands.

## Finding the Review File

1. If an argument is provided, use it as the review file path.
2. If no argument, search for `Review-*.md` files at the project root (exclude `*-supplementary.md`).
3. If multiple review files exist, use AskUserQuestion to let the user pick.
4. If none exist, tell the user and stop.

## Process

### 0. Establish a Clean Baseline

Before touching any code, verify the project's checks pass cleanly. Run the full validation suite (`make test-unit`, `make lint`, `make typecheck`, `make format`) and capture the results.

If any check fails, use AskUserQuestion to present the failures and ask:
- "Fix first" — resolve the pre-existing failures before starting review work (preferred — a clean baseline makes it obvious when a review fix introduces a regression)
- "Proceed anyway" — note the pre-existing failures and continue, accepting that validation will be noisier

A clean starting state is strongly preferred. When pre-existing failures exist and the user chooses to proceed, record the exact failure signatures (test names, lint rules, error messages) so they can be filtered out during per-finding validation in Step 3b.

### 1. Parse the Review

Read the review file and extract all findings from the **Findings** section. Each finding has:
- **ID**: The prefix code (e.g., `I0`, `I3`, `S0`, `C1`)
- **Title**: The finding name
- **File references**: Source file paths and line numbers
- **Fix description**: What the review recommends changing

Sort findings by implementation order — dependencies and risk:
1. Smallest/most isolated changes first (one-line fixes, no test changes)
2. Changes that modify behavior or APIs next
3. Structural refactors last (file moves, import rewrites)

Group related findings that must be applied together (e.g., if fixing a module structure also requires updating exports, those go in one step).

### 2. Create Tasks

Create a task for each finding (or group of related findings) using TaskCreate. This gives the user visibility into progress.

### 3. Iterate Through Findings

For each finding, follow this exact cycle:

#### 3a. Implement the Fix

- Mark the task as `in_progress`
- Read the affected source files before editing
- Make the code change described in the review
- If the fix changes behavior, update affected tests to match
- If the fix changes structure (moves files, renames modules), update all import references — use Grep to find every occurrence before editing

#### 3b. Validate

Run the project's test suite using make targets. Prefer the most targeted test command that covers the changed code. If it fails:
- Read the failure output
- Compare against the baseline from Step 0 — if these are the same pre-existing failures the user chose to proceed past, they're expected
- If the failure is new (not in the baseline), fix it before proceeding — this is a regression from the current change
- Re-run until the only failures are the pre-existing baseline ones (or none, if the baseline was clean)

#### 3c. Hand Off for Review

Use AskUserQuestion to notify the user. The question should include:
- Which finding was applied (by ID and title)
- Which files changed (brief summary)
- Test results (pass count)
- Options: "Yes, commit" / "Need changes"

If the user says "Need changes", wait for their feedback, apply it, re-validate, and ask again.

#### 3d. Wait for Staging

The user stages files themselves. Do NOT run `git add`. Instead:
- Check `git diff --cached --stat` to see if changes are staged
- If nothing is staged, use AskUserQuestion to remind the user to stage
- Once staged, proceed to commit

#### 3e. Commit

Write a targeted commit message based on the review finding:
- **Format**: Conventional commits (`fix(scope):`, `feat(scope):`, `refactor(scope):`)
- **Title**: Describes what changed, under 80 characters
- **Body**: Why the change was needed (reference the review finding ID)
- **Attribution**: `Assisted-by: Claude Code (<model-name>)` trailer
- Use the review ID (e.g., `Review: I3`) in the body so the commit links back to the finding

```bash
git commit -m "$(cat <<'EOF'
fix(scope): short description of what changed

Why this change was needed, referencing the review finding.

Review: I3

Assisted-by: Claude Code (Claude Opus 4.6)
EOF
)"
```

#### 3f. Mark Complete and Continue

- Mark the task as `completed`
- Move to the next finding

### 4. Final Verification

After all findings are applied:
- Run `make test-unit` (or equivalent full test suite)
- Run `make lint`
- Run `make typecheck`
- Run `make format`

Report the results. If any check fails and it's not pre-existing, investigate and fix before declaring done.

## Critical Rules

- **One finding per commit** — keeps changes reviewable and revertable
- **Never run `git add`** — the user controls staging
- **Never run `git push`** — the user controls when to push
- **Never change branches** — work on the current branch
- **Always validate before handoff** — don't present broken changes to the user
- **Always use AskUserQuestion at handoff** — the user has hooks that surface questions; plain text gets missed
- **Check staging before committing** — verify `git diff --cached` shows the expected files
- **Read before editing** — always read files before making changes
- **Use make targets** — prefer `make test-unit`, `make lint`, `make typecheck` over raw commands

## Edge Cases

- **Pre-existing failures**: Handled by Step 0. If the user chose "Proceed anyway", compare every failure against the recorded baseline signatures. New failures are regressions; baseline failures are expected noise.
- **Finding requires investigation**: If the review says "investigate X before fixing", do the investigation and report findings before making changes. Use AskUserQuestion to confirm the approach.
- **Finding is invalid**: If investigation reveals a finding doesn't apply (e.g., the code has already been fixed, or the finding was based on incorrect assumptions), use AskUserQuestion to tell the user and skip it.
