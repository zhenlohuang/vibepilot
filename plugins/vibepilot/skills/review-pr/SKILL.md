---
name: review-pr
description: Review the code in an existing GitHub pull request.
disable-model-invocation: true
argument-hint: "[PR number | PR URL]"
---

# /vibepilot:review-pr

Thin wrapper around Claude Code's built-in `/review` skill. Distinct from `/vibepilot:review`, which audits **local spec implementation** against `.vibepilot/spec/`.

## Preconditions

- `gh` installed and authenticated (`gh auth status`). On failure, surface `gh auth login` and stop.
- A PR resolvable from the argument or the current branch:
  - Empty argument → `gh pr view --json number,url,title,state` against the current branch. If no PR, tell the user to pass a number/URL or open one with `/vibepilot:create-pr` — don't fall back to local diff (that's `/vibepilot:review`).
  - Number or URL → `gh pr view <arg> --json number,url,title,state`. If `MERGED` / `CLOSED`, surface the state once and ask whether to proceed — historical review is fine but the user opts in.

## Argument parsing

- Empty → current branch's PR
- `123` → PR #123 in the current repo
- `https://github.com/owner/repo/pull/123` → that PR (cross-repo fine)
- Unparseable (e.g., `123abc`, a branch name) → ask the user once to clarify; don't guess.

## Steps

1. **Resolve the PR.** Apply argument parsing + preconditions. Print one confirmation line:

   ```
   Reviewing PR #<number> — <title>
   ```

   Don't paste body, diff, commit list, or reviewer roster.

2. **Delegate to the built-in `/review` skill** via the `Skill` tool with `skill: "review"`. Pass the resolved PR number or URL as `args`. The built-in skill owns the diff fetch, file reads, and grading — don't duplicate any of that here.

3. **Relay the result** verbatim, in the language it returned. Don't paraphrase, don't re-grade, don't paste the diff back.

## Constraints

- Match the user's conversation language for surface text (the "Reviewing PR …" line, precondition-failure messages, framing). Verdict tokens (`PASS`, `ADVISORY`, `BLOCK`) and command names stay canonical English.
- **Thin wrapper, period.** Don't read the PR diff, open source files, or write a review report yourself.
- **Sibling-skill boundary.** If the user clearly meant local audit (no PR exists, `tasks.md` has unchecked items, phrasing is about "my implementation"), redirect to `/vibepilot:review` and stop.
- Do not modify code, edit the PR title/body, or post comments.
- Do not create, merge, approve, request reviewers on, or change draft/ready state.
- Treat `.vibepilot/spec/` as read-only and, in this skill, untouched — don't supplement the PR review with local spec.
- On precondition failure, surface verbatim and stop. Don't retry, don't guess a different PR.
