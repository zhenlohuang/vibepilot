---
description: Review the code in an existing GitHub pull request.
argument-hint: "[PR number | PR URL]"
---

# /vibepilot:review-pr

Thin wrapper around Claude Code's built-in `/review` skill. Distinct from `/vibepilot:review`, which audits **local spec implementation**.

## Preconditions

- `gh` installed and authenticated (`gh auth status`). On failure, surface `gh auth login` and stop.
- A PR resolvable from the argument or the current branch:
  - Empty → `gh pr view --json number,url,title,state` against the current branch. If no PR, tell the user to pass a number/URL or open one with `/vibepilot:create-pr` — don't fall back to local diff (that's `/vibepilot:review`).
  - Number / URL (cross-repo OK) → `gh pr view <arg> --json number,url,title,state`. If `MERGED` / `CLOSED`, surface the state and ask whether to proceed.
  - Unparseable (e.g., `123abc`, a branch name) → ask the user once; don't guess.

## Flow

1. Resolve the PR, apply preconditions, print one confirmation line:

   ```
   Reviewing PR #<number> — <title>
   ```

   Don't paste body, diff, commit list, or reviewer roster.

2. **Delegate** to the built-in `/review` skill via the `Skill` tool (`skill: "review"`), passing the resolved PR number or URL as `args`. The built-in skill owns the diff fetch, file reads, and grading.

3. Relay the result verbatim, in the language it returned.

## Notes

- Thin wrapper, period — don't read the PR diff, open source files, or write a review report in the main thread.
- If the user clearly meant local audit (no PR exists, `tasks.md` has unchecked items, phrasing is about "my implementation"), redirect to `/vibepilot:review` and stop.
- Don't modify code, edit the PR title/body, post comments, merge, approve, request reviewers, or change draft/ready state.
- Verdict tokens (`PASS`, `ADVISORY`, `BLOCK`) and command names stay English.
