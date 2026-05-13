---
name: review
description: Audit the local implementation against the spec before committing.
disable-model-invocation: true
argument-hint: "[N | N-M | N,M,K | all]"
---

# /vibepilot:review

Audits **local spec implementation** against `.vibepilot/spec/`. For reviewing a GitHub PR, use `/vibepilot:review-pr`.

## Preconditions

- `.vibepilot/spec/{requirements,plan,tasks}.md` must all have substantive content. If any is missing or a placeholder, tell the user which step to run first and stop.
- `tasks.md` must contain at least one `- [x]`. If everything is still `- [ ]`, tell the user to run `/vibepilot:implement` first and stop.

## Argument parsing

- Empty or `all` → review every `- [x]` task
- `3` → only task 3
- `1-3` → tasks 1 through 3 inclusive
- `1,3,5` → exactly those tasks
- Unparseable → default to `all`, note the assumption when delegating.

## Steps

1. **Gather the diff** the subagent will need:
   - `git status --short` — uncommitted state
   - `git diff` — unstaged
   - `git diff --cached` — staged
   - `git log --oneline -20` — recent history

   If the user named a base branch ("review against `main`"), also include `git diff <base>...HEAD`.

   If `git status` is clean (everything already committed), use `git log <base>..HEAD --stat` and `git diff <base>...HEAD` (default `<base>` to `main`) to capture the committed change set.

   If the working tree has changes clearly unrelated to spec scope (files not in any task's `Files:` line), flag them as advisory when delegating — don't ignore.

2. **Delegate to `vibe-reviewer`** via the `Agent` tool with `subagent_type: "vibe-reviewer"`. Include:
   - Absolute path to `.vibepilot/spec/`
   - The review scope (specific task numbers or `all`)
   - The diff and log output from step 1
   - Instruction: read the three spec files, open whatever source files the diff touches, write the report to `.vibepilot/spec/review.md`. Return ONLY a summary under 200 words plus a verdict line: `PASS` / `ADVISORY` / `BLOCK`. The subagent must NOT modify source or any spec file other than `review.md`.

3. **Relay summary and verdict.**
   - `BLOCK` → list blocking issues briefly, suggest `/vibepilot:implement` again to fix.
   - `ADVISORY` → list suggestions briefly so the user can decide.
   - `PASS` → confirm succinctly, suggest committing if there's uncommitted work.

## Constraints

- Do not read source files in the main thread beyond what step 1 needs.
- Do not paste the full review report — point at `.vibepilot/spec/review.md`.
- Do not modify any code or spec files yourself.
