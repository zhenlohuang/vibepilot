---
name: review
description: Audit uncommitted changes against the spec before committing.
disable-model-invocation: true
argument-hint: "[optional scope]"
---

# /vibepilot:review

Audits **local spec implementation** against `.vibepilot/work/`. The skill is argumentless by default, but adapts to whatever scope the user describes. For reviewing a GitHub PR, use `/vibepilot:review-pr`.

## Preconditions

- `.vibepilot/work/{requirements,plan,tasks}.md` must all have substantive content. If any is missing or a placeholder, tell the user which step to run first and stop.
- Something to review — either uncommitted changes in the working tree, or commits ahead of the default base on the current branch. If neither (clean tree and no divergence), refuse with `Nothing to review — working tree clean and no commits ahead of <base>.` and stop.

## Default scope

No argument → review the uncommitted change set (staged + unstaged). If the working tree is clean, fall back to commits on the current branch versus the default base (`main`, falling back to `master`).

## Adapting to user input

If the user passes anything — task numbers, a branch name, file paths, a free-form description like "just the auth files" or "compare against develop" — interpret it and adjust the diff scope before delegating. Don't force the user into a specific arg syntax; trust their framing. Common reductions:

- **Task references** → narrow the review to the files those tasks touch (use `plan.md` / `tasks.md` to map task → files).
- **A branch reference** → use that branch as the base for `git diff <base>...HEAD`, regardless of working-tree state.
- **File / directory references** → narrow the gathered diff to those paths.

Resolution genuinely ambiguous → confirm via AskUserQuestion; otherwise proceed with the obvious reading. Validate any referenced task numbers exist in `tasks.md` — missing → ask `Skip missing and proceed` / `Cancel`.

## Steps

1. **Gather the diff** the subagent will need:
   - `git status --short` — uncommitted state
   - `git diff` — unstaged
   - `git diff --cached` — staged
   - `git log --oneline -20` — recent history

   If the resolved scope names a base branch (either default fallback or user-specified), gather `git log <base>..HEAD --stat` and `git diff <base>...HEAD` instead of (or in addition to) the working-tree diff. If `git status` is clean and no base was specified, default to `main` (then `master`) automatically.

   If the resolved scope names specific paths, restrict the `git diff` invocations to those paths.

   If the working tree has changes clearly unrelated to spec scope (files not in any task's `Files:` line), flag them as advisory when delegating — don't ignore.

2. **Delegate to `vibe-reviewer`** via the `Agent` tool with `subagent_type: "vibe-reviewer"`. Include:
   - Absolute path to `.vibepilot/work/`
   - The diff and log output from step 1, with a one-line note on the resolved scope (working tree vs `<base>..HEAD`, any path or task restriction)
   - Instruction: read the three spec files, open whatever source files the diff touches, write the report to `.vibepilot/work/review.md`. Return ONLY a summary under 200 words plus a verdict line: `PASS` / `ADVISORY` / `BLOCK`. The subagent must NOT modify source or any spec file other than `review.md`.

3. **Relay summary and verdict.**
   - `BLOCK` → list blocking issues briefly, suggest `/vibepilot:implement` again to fix.
   - `ADVISORY` → list suggestions briefly so the user can decide.
   - `PASS` → confirm succinctly, suggest `/vibepilot:commit` if there's uncommitted work.

## Constraints

- Do not read source files in the main thread beyond what step 1 needs.
- Do not paste the full review report — point at `.vibepilot/work/review.md`.
- Do not modify any code or spec files yourself.
