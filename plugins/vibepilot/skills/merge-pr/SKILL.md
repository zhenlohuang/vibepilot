---
name: merge-pr
description: Merge a GitHub pull request via `gh pr merge`, after confirming the PR is open, ready (not draft), and in a mergeable state. **Merge now or stop** — never schedules for later, never bypasses branch protection. The argument selects which PR — empty defaults to the current branch's open PR, a number like `123` targets PR #123 in this repo, a URL targets that PR. Asks which merge method to use (squash / merge commit / rebase, restricted to what the repo allows) and whether to delete the branch afterwards. Trigger whenever the user wants an existing PR merged — `/vibepilot:merge-pr`, "merge this PR", "merge the pull request", "land the PR", "merge PR #123", "ship the PR", "merge it in", "go ahead and merge", "ready to merge", "合并这个PR", "把PR合了", "合并一下", "合PR", "把PR合并了", "合并到main", and similar — including when the user references a PR by number or URL, or simply says "merge" with a PR clearly in scope. Especially valid right after `/vibepilot:review-pr` returns PASS or the user has gathered the approvals they wanted. Do NOT trigger for opening a new PR (`/vibepilot:create-pr`), reviewing or commenting on a PR (`/vibepilot:review-pr`), approving a PR, requesting reviewers, closing without merging, reopening a closed PR, force-merging via admin override, scheduling an auto-merge for later, switching draft/ready state, editing the PR title/body, or auditing local spec implementation (`/vibepilot:review`) — those are different intents.
---

# vibepilot:merge-pr — merge a pull request

Merge a GitHub pull request via `gh pr merge`, after confirming the PR is in a mergeable state. This runs in the main thread because two decisions need user back-and-forth — which merge method to use (squash / merge commit / rebase) and whether to delete the branch — and the final action is irreversible enough on a shared remote that every failure mode must surface to the user, not be silently retried.

The skill is **generic** with respect to `.vibepilot/spec/` — merging is post-PR, so the spec has no role here. The argument selects the PR; without one, the current branch's open PR is used.

## Preconditions

Check these first. If any fails, surface the state and stop — don't try to recover automatically, because "merge a PR" stops being well-defined once the target or its state is ambiguous:

- `gh` must be installed and authenticated (`gh auth status` succeeds). On failure, surface `gh auth login` as the fix and stop. Merging against GitHub requires `gh`; this skill cannot substitute a local-only path.
- A PR must be resolvable from the argument or the current branch.
  - Empty argument → `gh pr view --json ...` against the current branch. If no PR exists, tell the user to either pass a PR number/URL explicitly or open one with `/vibepilot:create-pr` first. Do not silently fall back to anything else.
  - Number or URL → `gh pr view <arg> --json ...` to confirm it exists.
- The PR state must be `OPEN`. If `MERGED`, surface its URL once and stop — already merged. If `CLOSED`, surface and stop — reopening is a separate intent.
- The PR must not be a draft (`isDraft = false`). If it is, tell the user to mark it ready first (`gh pr ready`) and stop. Draft PRs are explicitly not ready to merge.

## Argument parsing

The argument selects the PR:

- Empty → current branch's open PR (detected via `gh pr view`)
- A number `123` → PR #123 in the current repo
- A URL `https://github.com/owner/repo/pull/123` → that PR (cross-repo is fine; `gh` handles it)

If the input is unparseable (e.g., `123abc`, a branch name), ask the user once which PR they mean — don't guess.

## Language

Match the user's conversation language for the surface text you write yourself — the "Merging PR …" confirmation line, AskUserQuestion prompts, mergeability concern summaries, and any precondition-failure or post-merge messages. Keep GitHub state tokens (`OPEN`, `MERGED`, `CLOSED`, `CLEAN`, `BLOCKED`, `BEHIND`, `DIRTY`, `UNSTABLE`), command names (`gh`, `git`), and merge-method labels (`squash`, `merge`, `rebase`) in canonical English so behavior and downstream tooling remain deterministic.

## Steps

1. **Resolve the PR + fetch full state.** Run in parallel:
   - `gh pr view <ref> --json number,url,title,state,isDraft,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,headRefName,baseRefName,headRepositoryOwner`
   - `gh repo view --json mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed,deleteBranchOnMerge`

   Apply preconditions on the result.

2. **Print one confirmation line** so the user sees exactly what's being merged:

   ```
   Merging PR #<number> — <title> (<headRefName> → <baseRefName>)
   ```

   Just this one line. Do not paste the PR body, the diff, the commit list, or the full check/review roster — step 3 surfaces only what blocks the merge.

3. **Inspect mergeability and surface concerns.** Read `mergeable` + `mergeStateStatus` + `reviewDecision` + `statusCheckRollup`:
   - `mergeable = MERGEABLE` and `mergeStateStatus = CLEAN` (or `HAS_HOOKS`) → green; proceed to step 4.
   - `mergeable = CONFLICTING` (or `mergeStateStatus = DIRTY`) → stop. Conflicts must be resolved by the user; this skill won't initiate a merge or rebase.
   - `mergeStateStatus = BLOCKED` (failing required checks, missing required reviews, or `reviewDecision = CHANGES_REQUESTED`) → print one short summary line per failing check or missing review (cap at ~5 lines), then ask via AskUserQuestion: `Stop and let you resolve (Recommended)` / `Attempt merge anyway`. If the user chooses "attempt anyway", let `gh` enforce branch protection — do **not** add `--admin`.
   - `mergeStateStatus = BEHIND` → tell the user the PR is behind base; ask: `Stop — update branch separately (Recommended)` / `Attempt merge anyway`. Don't force-update or rebase from this skill.
   - `mergeStateStatus = UNSTABLE` (required checks pass, some non-required failing) → ask: `Proceed — non-required checks are failing (Recommended)` / `Stop and investigate`.
   - `statusCheckRollup` has `PENDING` checks → ask: `Wait — let checks finish (Recommended)` / `Proceed without waiting`. Do not auto-poll; the user re-invokes the skill if they want to wait and retry.
   - `mergeable = UNKNOWN` → wait one beat, re-run the `gh pr view` once; if still UNKNOWN, surface and stop.

4. **Ask which merge method to use.** Build the option list from the repo's allowed methods (`mergeCommitAllowed`, `squashMergeAllowed`, `rebaseMergeAllowed`):
   - If only one is enabled, skip the prompt entirely and use it; print one line stating which method (so the user isn't surprised by a choice they didn't make).
   - If two or more are enabled, ask via AskUserQuestion with only the allowed methods as options. Recommend `Squash and merge` first when squash is among the allowed set — it's the most common modern default for feature branches and produces a single commit on the base branch. If squash is disallowed, recommend `Create a merge commit` before `Rebase and merge`. The user can always pick another.
   - Do not try to infer the repo's "real" default by inspecting `git log --merges` or recent merged PRs — that heuristic is fragile (a squash-default repo has no merge commits to learn from), and the prompt is cheap.

5. **Ask whether to delete the branch after merge.**
   - **Cross-repo (fork) PRs** → never offer to delete the head branch. Detect by comparing the PR's `headRepositoryOwner.login` to the target repo's owner login (from `gh repo view --json owner`); if they differ, the head branch lives on someone else's fork and we can't and shouldn't delete it. Skip the prompt entirely.
   - **Same-repo PRs with `deleteBranchOnMerge = true`** → remote deletion is automatic on merge; skip the prompt and note this once so the user isn't surprised.
   - **Same-repo PRs otherwise** → ask via AskUserQuestion: `Delete the branch (Recommended)` / `Keep the branch`. The recommendation assumes a feature-branch workflow where the head branch's life ends at merge; the user can override.
   - Heads up the user about the local-deletion semantic if they choose to delete: `gh pr merge --delete-branch` removes **both** the remote branch and the local branch (gh handles the switch to the base branch automatically if you're currently on the head branch). This is by design — one short sentence in the surface text is enough.

6. **Merge.** Run `gh pr merge <ref>` with exactly the flags from steps 4 and 5:
   - One of `--merge` / `--squash` / `--rebase` from step 4.
   - `--delete-branch` if chosen in step 5 (omit if the repo auto-deletes, or for cross-repo PRs).
   - Do not pass `--admin` (no branch-protection bypass), `--auto` (auto-merge-when-ready is a different intent), or `--subject` / `--body` (let `gh` use the PR title/body that `/vibepilot:create-pr` already shaped — rewriting at merge time creates a second narrative that diverges from the PR).

   On any `gh pr merge` failure (auth scope, branch protection, required reviews, status checks, hook), surface its output verbatim and stop. Do not retry with different flags — almost every failure here requires the user to handle something out-of-band.

7. **Report.**
   - Run `gh pr view <ref> --json number,url,state,mergedAt -q '"#\(.number) \(.state) \(.url)"'` and print that single line.
   - **Local-branch cleanup hint.** Only if `--delete-branch` was **not** passed (so `gh` left the local checkout alone) **and** the user's current local branch is the head branch (`git symbolic-ref --short HEAD` equals the PR's `headRefName`), append one suggestion line so they can clean up if they want — but do **not** run it yourself:

     ```
     Local branch <name> is now merged on the remote. Clean up with: git switch <base> && git branch -d <name>
     ```

     If `--delete-branch` was passed, skip this hint entirely — `gh` already handled local + remote, and a stale suggestion would just confuse the user.
   - That's the entire report. Do not paste the merge commit, the resulting diff, or any other follow-on output.

## Constraints

These guardrails exist because the skill takes a destructive, visible action on a shared remote — merging a PR is hard to undo and seen by everyone watching the repo. Keep them tight:

- **No admin override.** Never pass `--admin` to `gh pr merge`. If branch protection blocks the merge, the user resolves it out-of-band — they may not even be an admin, the protection may be intentional, and silently bypassing it on their behalf is unsafe.
- **No auto-merge.** Do not pass `--auto`. Scheduling a merge for later is a different intent — `/vibepilot:merge-pr` means "merge now, or stop and let me fix it".
- **No customizing the merge commit message.** Do not pass `--subject` or `--body`. The PR title/body (already shaped by `/vibepilot:create-pr`) is the source of truth; rewriting at merge time creates a second narrative that diverges from the PR.
- **No approving, requesting reviewers, posting comments, switching draft/ready state, or editing the PR title/body.** Those are different intents — bow out so the user gets to ask for them deliberately.
- **No retries on `gh` failure.** Auth, rate limit, branch protection, required reviews, hook — surface verbatim and stop. Retrying with different flags almost always escalates a recoverable situation into a confusing one.
- **No force-pushing or rebasing the PR branch.** If `mergeStateStatus = BEHIND` or `DIRTY`, this skill stops. Updating the branch is a separate, potentially destructive intent — the user runs it deliberately.
- **Stay in the main thread.** Do not invoke the `Agent` tool — the mergeability concern surfacing, the merge-method choice, and the delete-branch prompt are inherently interactive and a subagent can't carry them.
- **Treat `.vibepilot/spec/` as read-only and untouched.** Merging is an after-PR action; the spec has no role here, and reading it would only invite drift between the spec and what's actually being merged.
- **Do not run `git branch -d` yourself.** The legitimate path to local-branch deletion is `gh pr merge --delete-branch`, chosen explicitly by the user in step 5 — `gh` handles the worktree switch and refuses if local has unmerged changes. Surfacing the cleanup hint in step 7 is fine; running the `git` command for the user is not. Local branches can hold uncommitted work or be attached to worktrees this skill can't see.
- **Do not paste the PR body, the full diff, the full check list, or the full review thread into the conversation.** The one-line identification in step 2, the targeted concern summaries in step 3, and the `gh pr view` line in step 7 are the entire report. The user already has the PR rendered on github.com.
- **No AI-attribution trailers anywhere.** This skill does not write commit messages or PR bodies (constraint above), but if any surface text it does write is ever displayed, do not append `🤖 Generated with …` or `Co-Authored-By: Claude …`.
