---
name: review
description: Audit the code changes produced for the current spec against `requirements.md` / `plan.md` / `tasks.md` and write a report to `.vibepilot/spec/review.md`. Delegates to the `vibe-reviewer` subagent. Trigger when the user runs `/vibepilot:review` (optionally with a task number, range, or comma list), asks to audit / verify / review the implementation, or wants a pass over the changes before committing — even when they only say "did I get it right" or "check my work".
---

# vibepilot:review — audit the implementation

You are running a review pass over the code changes produced for the current spec. The heavy work — reading diffs, opening source files, cross-checking acceptance criteria — is delegated to the `vibe-reviewer` subagent. Doing it inline would dump diffs and source into the main thread for no benefit.

## Preconditions

- `.vibepilot/spec/{requirements,plan,tasks}.md` must all exist with substantive content. If any is missing or still a placeholder, tell the user which step to run first and stop.
- `tasks.md` must contain at least one `- [x]` item — otherwise there's nothing implemented to review. If everything is still `- [ ]`, tell the user to run `/vibepilot:implement` first and stop.

## Argument parsing

The argument selects review scope:

- Empty or `all` → review every `- [x]` task
- A single number `3` → only task 3
- A range `1-3` → tasks 1 through 3 inclusive
- A comma-separated list `1,3,5` → exactly those tasks

If the input is unparseable, default to `all` and note the assumption in the report.

## Steps

1. **Gather the diff** the subagent will need:
   - `git status --short` — uncommitted state at a glance
   - `git diff` — unstaged changes
   - `git diff --cached` — staged changes (the user may have already staged work)
   - `git log --oneline -20` — recent history for context

   If the user explicitly mentioned a base branch (e.g., "review against `main`"), also run `git diff <base>...HEAD` and include that.

   **If `git status` shows no uncommitted changes** (everything already committed), the implementation is fully landed in commits. In that case run `git log <base>..HEAD --stat` (default `<base>` to `main`) and `git diff <base>...HEAD` to capture the committed change set for the review.

   **If the working tree has changes that are clearly unrelated to the current spec** (files touched that aren't in any task's `Files:` line), note that briefly when delegating — the reviewer should focus on spec-scope files but flag clearly-unrelated changes as advisory rather than ignore them.

2. **Delegate to `vibe-reviewer`** via the `Agent` tool with `subagent_type: "vibe-reviewer"`. The prompt should include:
   - The absolute path to `.vibepilot/spec/`
   - The review scope (specific task numbers or `all`)
   - The diff and log output from step 1 (paste it in)
   - An instruction: read the three spec files, open whatever source files the diff touches, and write the report to `.vibepilot/spec/review.md`. Return ONLY a short summary (under 200 words) plus a verdict line: `PASS` / `ADVISORY` / `BLOCK`.
   - A reminder: the subagent must NOT modify source code or any spec file other than `review.md`.

3. **Relay the summary and verdict** to the user:
   - `BLOCK` → list the blocking issues briefly and suggest running `/vibepilot:implement` again to fix them.
   - `ADVISORY` → list the suggestions briefly so the user can decide which to act on.
   - `PASS` → confirm succinctly and suggest committing if there's uncommitted work.

## Constraints

- Do **not** read source files or open diffs in the main thread beyond what step 1 needs. The point of delegating is to keep this thread clean for the verdict conversation.
- Do **not** paste the full review report into the conversation — point the user at `.vibepilot/spec/review.md`.
- Do **not** modify any code or spec files yourself.
