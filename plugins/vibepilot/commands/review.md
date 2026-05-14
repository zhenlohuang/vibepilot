---
description: Audit uncommitted changes against the spec before committing.
argument-hint: "[optional scope]"
---

# /vibepilot:review

Audits **local spec implementation** against `.vibepilot/work/`. For reviewing a GitHub PR, use `/vibepilot:review-pr`.

## Preconditions

- `.vibepilot/work/{requirements,plan,tasks}.md` must all be substantive. Otherwise tell the user which step to run first and stop.
- Something to review (dirty tree or commits ahead of base). If neither: `Nothing to review — working tree clean and no commits ahead of <base>.` and stop.

## Flow

1. **Resolve the scope.** Probe state first:
   - `git status --short` — is the tree dirty?
   - `git rev-list --count <base>..HEAD` against `main` (then `master`) — is HEAD ahead?

   Then:
   - User passed an arg (task numbers, branch, paths, free-form) → trust it. Common reductions: task refs narrow to those tasks' `Files:` paths; a branch ref becomes the diff base; path refs narrow the diff. Validate any task numbers exist (missing → `Skip missing` / `Cancel`).
   - No arg, both scopes available → AskUserQuestion: `Uncommitted changes (X files) (Recommended)` / `Branch vs <base> (N commits)`. `Other` text routes through the user-input path above.
   - No arg, only one available → use it, print a one-line note (`Reviewing uncommitted changes` / `Reviewing branch vs <base>`).

   Capture the resolved scope as `working-tree`, `<base>..HEAD`, or a path/task-restricted variant.

2. **Gather the diff** for the resolved scope:
   - `working-tree` → `git status --short`, `git diff`, `git diff --cached`, `git log --oneline -20`.
   - `<base>..HEAD` → `git log <base>..HEAD --stat`, `git diff <base>...HEAD`, `git log --oneline -20`.
   - Path/task-restricted → add `-- <paths>`.

   If the tree has changes clearly outside any task's `Files:` line, flag them as advisory when delegating.

3. **Delegate to `vibe-reviewer`** (`subagent_type: "vibe-reviewer"`) with:
   - Absolute path to `.vibepilot/work/`
   - The diff and log output, plus a one-line scope note
   - Instruction: read the three spec files, open whatever source files the diff touches, write the report to `review.md`, return a summary under 200 words plus a verdict line: `PASS` / `ADVISORY` / `BLOCK`. The subagent must NOT modify source or any spec file other than `review.md`.

4. Relay summary and verdict:
   - `BLOCK` → list blocking issues briefly; suggest `/vibepilot:implement` again.
   - `ADVISORY` → list suggestions briefly so the user can decide.
   - `PASS` → confirm; suggest `/vibepilot:commit` if there's uncommitted work.

## Notes

- Don't read source files in the main thread beyond what step 2 needs.
- Don't paste the full review report — point at `.vibepilot/work/review.md`.
- Don't modify any code or spec files in the main thread.
