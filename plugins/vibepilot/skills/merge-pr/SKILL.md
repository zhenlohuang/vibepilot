---
name: merge-pr
description: Merge a GitHub pull request.
disable-model-invocation: true
argument-hint: "[PR number | PR URL]"
---

# /vibepilot:merge-pr

Generic with respect to `.vibepilot/spec/` — merging is post-PR, spec has no role.

## Preconditions

Surface state and stop if any fails:

- `gh` installed and authenticated (`gh auth status`). On failure, surface `gh auth login` and stop.
- A PR resolvable from the argument or the current branch:
  - Empty → `gh pr view --json …` against current branch. If no PR, tell the user to pass a number/URL or open one with `/vibepilot:create-pr`. No silent fallback.
  - Number or URL → `gh pr view <arg> --json …` to confirm.
- PR state is `OPEN`. If `MERGED`, surface URL and stop. If `CLOSED`, surface and stop — reopening is a separate intent.
- PR is not draft (`isDraft = false`). If it is, tell the user to mark it ready (`gh pr ready`) and stop.

## Argument parsing

- Empty → current branch's open PR
- `123` → PR #123 in the current repo
- `https://github.com/owner/repo/pull/123` → that PR (cross-repo fine)
- Unparseable → ask the user once which PR; don't guess.

## Steps

1. **Resolve + fetch state** (parallel):
   - `gh pr view <ref> --json number,url,title,state,isDraft,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,headRefName,baseRefName,headRepositoryOwner`
   - `gh repo view --json mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed,deleteBranchOnMerge,owner`

   Apply preconditions.

2. **Print one confirmation line:**

   ```
   Merging PR #<number> — <title> (<headRefName> → <baseRefName>)
   ```

3. **Inspect mergeability** (`mergeable` + `mergeStateStatus` + `reviewDecision` + `statusCheckRollup`):
   - `MERGEABLE` + `CLEAN` (or `HAS_HOOKS`) → proceed.
   - `CONFLICTING` (or `DIRTY`) → stop. Conflicts are the user's to resolve.
   - `BLOCKED` (failing required checks, missing required reviews, `CHANGES_REQUESTED`) → print up to 5 short summary lines per failing check / missing review, then ask: `Stop and let you resolve (Recommended)` / `Attempt merge anyway`. **Never add `--admin`** — let `gh` enforce branch protection.
   - `BEHIND` → ask: `Stop — update branch separately (Recommended)` / `Attempt merge anyway`. Don't force-update or rebase.
   - `UNSTABLE` (required checks pass, non-required failing) → ask: `Proceed — non-required checks are failing (Recommended)` / `Stop and investigate`.
   - `statusCheckRollup` has `PENDING` checks → ask: `Wait — let checks finish (Recommended)` / `Proceed without waiting`. Don't auto-poll.
   - `UNKNOWN` → wait one beat, re-run `gh pr view` once; if still UNKNOWN, surface and stop.

4. **Ask merge method** built from the repo's allowed set (`mergeCommitAllowed`, `squashMergeAllowed`, `rebaseMergeAllowed`):
   - Only one enabled → skip the prompt, use it, print one line stating the choice.
   - Two+ enabled → AskUserQuestion. Recommend `Squash and merge` first when squash is allowed; otherwise `Create a merge commit` before `Rebase and merge`.
   - Don't infer the repo's "real default" from `git log --merges` — fragile and unnecessary.

5. **Ask whether to delete the branch.**
   - **Cross-repo (fork) PR** → never offer. Detect by comparing `headRepositoryOwner.login` to the repo's owner login; if different, skip the prompt.
   - **Same-repo + `deleteBranchOnMerge = true`** → auto on merge; skip the prompt and note once.
   - **Same-repo otherwise** → AskUserQuestion: `Delete the branch (Recommended)` / `Keep the branch`. Heads-up: `gh pr merge --delete-branch` removes **both** remote and local branches (`gh` handles the worktree switch).

6. **Merge.** `gh pr merge <ref>` with exactly:
   - One of `--merge` / `--squash` / `--rebase` from step 4.
   - `--delete-branch` if chosen (omit if repo auto-deletes or cross-repo).
   - **Never** `--admin`, `--auto`, `--subject`, `--body`.

   On failure, surface output verbatim and stop. Don't retry with different flags.

7. **Report.**
   - `gh pr view <ref> --json number,url,state,mergedAt -q '"#\(.number) \(.state) \(.url)"'`
   - **Local-branch cleanup hint** only if `--delete-branch` was NOT passed AND current local branch is the head branch (`git symbolic-ref --short HEAD` equals `headRefName`):

     ```
     Local branch <name> is now merged on the remote. Clean up with: git switch <base> && git branch -d <name>
     ```

     Don't run it yourself. If `--delete-branch` was passed, skip this hint.

## Constraints

- Match the user's conversation language for the confirmation line, AskUserQuestion prompts, mergeability concerns, precondition-failure messages, and post-merge text. State tokens (`OPEN`, `MERGED`, `CLOSED`, `CLEAN`, `BLOCKED`, `BEHIND`, `DIRTY`, `UNSTABLE`), merge-method labels (`squash`, `merge`, `rebase`), and command names stay canonical English.
- **No admin override.** Never pass `--admin`. Branch protection blocks → user resolves out-of-band.
- **No auto-merge.** Never pass `--auto`. Scheduling for later is a different intent.
- **No customizing the merge commit message.** Never pass `--subject` / `--body`. PR title/body is the source of truth.
- **No approving, commenting, requesting reviewers, switching draft/ready, or editing the PR title/body.** Different intents.
- **No retries on `gh` failure.** Surface verbatim and stop.
- **No force-pushing or rebasing the PR branch.** `BEHIND` / `DIRTY` → stop.
- Stay in the main thread — do not invoke the `Agent` tool.
- Treat `.vibepilot/spec/` as read-only and untouched.
- **Do not run `git branch -d` yourself.** Use `--delete-branch` if the user opted in (step 5); the hint in step 7 is text-only.
- Do not paste PR body, full diff, full check list, or full review thread — the step 2 / 3 / 7 outputs are the entire report.
- **No AI-attribution trailers** in any surface text.
