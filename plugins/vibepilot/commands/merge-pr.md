---
description: Merge a GitHub pull request.
argument-hint: "[PR number | PR URL]"
---

# /vibepilot:merge-pr

Generic with respect to `.vibepilot/work/` — merging is post-PR. Stays in the main thread — no `Agent` calls.

## Preconditions

Surface and stop on any failure:

- `gh` installed and authenticated (`gh auth status`). On failure, surface `gh auth login`.
- A PR resolvable from the argument or the current branch:
  - Empty → `gh pr view --json …` against current branch. If no PR, tell the user to pass a number/URL or open one with `/vibepilot:create-pr`.
  - `123` / `https://github.com/owner/repo/pull/123` (cross-repo OK) → `gh pr view <arg> --json …`.
  - Unparseable → ask once; don't guess.
- PR state is `OPEN`. If `MERGED`/`CLOSED`, surface URL and stop.
- PR is not draft. Otherwise tell the user to run `gh pr ready` and stop.

## Flow

1. **Resolve + fetch state** in parallel:
   - `gh pr view <ref> --json number,url,title,state,isDraft,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,headRefName,baseRefName,headRepositoryOwner`
   - `gh repo view --json mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed,deleteBranchOnMerge,owner`

   Apply preconditions.

2. Print one confirmation line: `Merging PR #<number> — <title> (<headRefName> → <baseRefName>)`.

3. **Inspect mergeability** (`mergeable` + `mergeStateStatus` + `reviewDecision` + `statusCheckRollup`):
   - `MERGEABLE` + `CLEAN` (or `HAS_HOOKS`) → proceed.
   - `CONFLICTING` (or `DIRTY`) → stop. Conflicts are the user's to resolve.
   - `BLOCKED` (failing required checks, missing required reviews, `CHANGES_REQUESTED`) → print up to 5 short lines per failing check / missing review, then AskUserQuestion: `Stop and let you resolve (Recommended)` / `Attempt merge anyway`. **Never add `--admin`.**
   - `BEHIND` → AskUserQuestion: `Stop — update branch separately (Recommended)` / `Attempt merge anyway`. Don't force-update or rebase.
   - `UNSTABLE` (required pass, non-required failing) → AskUserQuestion: `Proceed — non-required checks failing (Recommended)` / `Stop and investigate`.
   - `statusCheckRollup` has `PENDING` → AskUserQuestion: `Wait — let checks finish (Recommended)` / `Proceed without waiting`. Don't auto-poll.
   - `UNKNOWN` → wait ~3 s, re-run `gh pr view` once; if still UNKNOWN, surface and stop.

4. **Ask merge method** from the repo's allowed set:
   - Only one enabled → use it, print one line stating the choice.
   - Two+ enabled → AskUserQuestion. Recommend `Squash and merge` first when squash is allowed; otherwise `Create a merge commit` before `Rebase and merge`.
   - Don't infer the "real default" from `git log --merges`.

5. **Ask whether to delete the branch.**
   - Cross-repo (fork) PR → never offer (compare `headRepositoryOwner.login` to the repo's owner; if different, skip).
   - Same-repo + `deleteBranchOnMerge = true` → auto on merge; note once and skip the prompt.
   - Same-repo otherwise → AskUserQuestion: `Delete the branch (Recommended)` / `Keep the branch`. Heads-up: `--delete-branch` removes both remote and local branches.

6. **Merge.** `gh pr merge <ref>` with exactly one of `--merge` / `--squash` / `--rebase`, plus `--delete-branch` if chosen. **Never** `--admin`, `--auto`, `--subject`, `--body`. On failure, surface output verbatim and stop.

7. **Report.**
   - `gh pr view <ref> --json number,url,state,mergedAt -q '"#\(.number) \(.state) \(.url)"'`
   - Local-branch cleanup hint only if `--delete-branch` was NOT passed AND current branch equals `headRefName`:

     ```
     Local branch <name> is now merged on the remote. Clean up with: git switch <base> && git branch -d <name>
     ```

     Don't run it yourself.

## Notes

- Confirmation line, AskUserQuestion prompts, mergeability concerns, precondition failures, and post-merge text match the user's conversation language. State tokens (`OPEN`, `MERGED`, `CLEAN`, `BLOCKED`, …), merge methods (`squash`/`merge`/`rebase`), and command names stay English.
- **No admin override.** Never pass `--admin`. Branch protection blocks → user resolves out-of-band.
- **No auto-merge.** Never pass `--auto`.
- **No customizing the merge commit message.** Never pass `--subject` / `--body`.
- Don't approve, comment, request reviewers, switch draft/ready, or edit the PR title/body — different intents.
- No retries on `gh` failure. No force-pushing or rebasing the PR branch.
- Treat `.vibepilot/work/` as read-only and untouched.
- Don't run `git branch -d` yourself — use `--delete-branch` or print the hint as text.
- Step 2 / 3 / 7 outputs are the entire report — no PR body, full diff, or full review thread.
- **No AI-attribution trailers** in any surface text.
