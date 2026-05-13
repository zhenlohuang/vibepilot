---
name: create-pr
description: Open a GitHub pull request for the current branch via `gh pr create`. **One PR per feature, not per task** — when a `.vibepilot/spec/` is active, the PR represents the whole spec, with title from `# Title`, Summary distilled from `## Background` (+ `plan.md` approach), and Test plan from the acceptance criteria; `tasks.md` informs completion state only, never the body. Without a spec, title and body synthesize from `git log <base>..HEAD`. Auto-delegates to `/vibepilot:commit` if uncommitted changes exist, and pushes the branch (`git push -u origin <branch>` when no upstream, `git push` when ahead) before creating the PR. Trigger whenever the user wants a PR opened — `/vibepilot:create-pr`, "open a PR", "create a pull request", "make a PR", "open PR for review", "send this for review", "ship it for review", "提交PR", "提个PR", "开个PR", "发个PR", and similar — including standalone use without an active spec. Especially valid after `/vibepilot:review` reports PASS, or after `/vibepilot:commit` lands the final task of a feature. Do NOT trigger for reviewing or commenting on an existing PR, merging a PR, editing a PR's description or title after creation, switching draft/ready state, requesting reviewers, or creating a PR against a non-default base branch — those are different intents.
---

# vibepilot:create-pr — open a pull request

Open a GitHub pull request for the current branch via `gh pr create`, after first ensuring the branch is committed and pushed. This runs in the main thread because three decisions need user back-and-forth — what to do about untracked files when `commit` is auto-invoked, whether to open as Draft or Ready, and how to respond to any push or `gh` failure. The title and body are bounded text, so reading them inline doesn't pollute the thread.

The skill is **generic**: it works with or without an active `.vibepilot/spec/`. Granularity is **one PR per feature** — when a spec is active, the PR represents the whole spec. `requirements.md` (the feature description and acceptance criteria) and `plan.md` (the approach) drive the title and body; `tasks.md` is an internal implementation breakdown and is consulted only for completion state, never enumerated into the body. Without a spec, both title and body are synthesized at a feature level from the commit log on the branch.

## Preconditions

Check these first. If any fails, surface the state and stop — don't try to recover automatically, because "open a PR" stops being well-defined once the repo or PR state is ambiguous:

- A git repository must exist (`git rev-parse --git-dir` succeeds). If not, tell the user this isn't a git repo and stop.
- `gh` must be installed and authenticated. Run `gh auth status`; on failure, surface `gh auth login` as the fix and stop. The skill cannot recover from auth issues on the user's behalf.
- The repo must not be mid-rebase, mid-merge, mid-cherry-pick, or in detached HEAD. The user should resolve that state first; this skill won't attempt it.
- The current branch must not be the repo's default branch (typically `main` / `master` — detect via `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`). If it is, tell the user to switch to a feature branch and stop — opening a PR from the default branch against itself isn't a meaningful action.
- An open PR for this branch must not already exist (`gh pr list --head <branch> --state open --json url,number,isDraft`). If one does, surface its URL and stop. Updating or reopening an existing PR is a different intent; duplicating is destructive.

## Language

Match the user's conversation language for the **title** and **body** of the PR (and, when a spec is active, the language of `requirements.md`). Keep the body section headers (`## Summary`, `## Test plan`) and any conventional tokens (ticket-ID prefixes, scope tags) in canonical English so GitHub renders them consistently and downstream tooling can parse them deterministically.

## Steps

1. **Gather repo + PR state.** Run these in parallel and inspect:
   - `git rev-parse --git-dir` — repo check
   - `gh auth status` — auth check
   - `git status --short` — does anything need committing?
   - `git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD` — current branch / detect detached HEAD
   - `gh repo view --json defaultBranchRef -q .defaultBranchRef.name` — base branch
   - `git status --porcelain=v2 --branch` — upstream tracking + ahead/behind line

   Once the branch and base are known, run these in parallel:
   - `gh pr list --head <branch> --state open --json url,number,isDraft`
   - `git rev-list --left-right --count <base>...HEAD` — exact ahead/behind counts
   - `git log <base>..HEAD --oneline` — commit shape on this branch

   Apply the preconditions. If `.git/rebase-*`, `.git/MERGE_HEAD`, or `.git/CHERRY_PICK_HEAD` is referenced in status, stop and ask the user to resolve.

2. **Handle uncommitted changes by delegating to `commit`.** If `git status --short` is non-empty, do not try to open a PR over uncommitted work — the user has explicitly opted into this coupling. Invoke `/vibepilot:commit` if your harness exposes skill invocation; otherwise read `plugins/vibepilot/skills/commit/SKILL.md` and follow its Steps inline. After it completes, re-run step 1 from the top. If the user aborts inside `commit` and the tree is still dirty, stop — a PR over uncommitted work is the case this skill refuses to handle silently.

3. **Surface "behind base" as a warning, not a block.** If the rev-list shows the branch is behind base, print one line like `Branch is N commits behind <base> — rebase/merge separately if you want a linear PR.` and continue. Rebasing or merging is a separate, potentially destructive intent; this skill won't initiate it. GitHub will compute the diff against the base correctly either way.

4. **Push the branch if needed.**
   - No upstream → `git push -u origin <branch>`.
   - Upstream exists and local is ahead → `git push` (no flags).
   - Already up-to-date → skip.
   - On any push failure, surface git's output verbatim and stop. Do **not** retry with `--force` or `--force-with-lease`, and do not change the upstream. A `fetch first` rejection is the same "behind base" situation — surface and stop so the user decides.

5. **Read context for title and body.**
   - **If `.vibepilot/spec/requirements.md` exists with substantive content** → this PR represents the whole feature. Read at the feature level:
     - `# Title` → strong title candidate.
     - `## Background` paragraph → the primary input for the Summary section (the *why* and the *what*, user-facing).
     - The acceptance-criteria section → the primary input for the Test plan section.
     - `plan.md` (if present, particularly the high-level approach/notes) → flavor for the Summary, not its skeleton.
     - `tasks.md` → consulted **only** for completion state. Count `- [ ]` vs `- [x]`. **Do not enumerate task titles into the body** — a task list is the developer's implementation breakdown; PR readers want to know what the feature *does*, not how it was sliced for work. If some tasks are still `- [ ]`, note it (`Partial feature: M/N tasks incomplete`) and let step 7 bias toward Draft.
     - `.vibepilot/spec/review.md` if present → a `PASS` outcome means the default for step 7 is Ready; anything else means Draft.
   - **Otherwise** → use the `git log <base>..HEAD --oneline` you already gathered to infer the feature shape across all commits on this branch — think of it as reconstructing the missing `## Background`. If the log is thin (one or two terse messages), supplement once with `git diff <base>..HEAD --stat`. Do not read the full diff — `--stat` is enough to choose a feature-level title. The Summary should still describe the feature, not list commits.
   - **In both cases** → if the repo's recent merged PRs use a clear convention (ticket-ID prefix like `[ABC-123]`, scope tag like `feat:`), mirror that accent. Run `gh pr list --state merged --limit 10 --json title -q '.[].title'` once if unsure. Don't invent a convention that isn't there.

6. **Synthesize the title and body** using the format below. Do **not** ask the user to approve them before creating — they chose this skill to delegate that judgment. Surface a one-line preview in the report instead.

7. **Ask Draft vs Ready** via AskUserQuestion. Order the options so the recommended one is first:
   - When `review.md` shows `PASS` **and** all tasks in `tasks.md` are `- [x]` (or no spec, with a small self-contained change) → `Open as Ready (Recommended)` / `Open as Draft`.
   - Otherwise (no `review.md`, non-PASS review, **any unchecked task in the spec**, or substantial scope without review) → `Open as Draft (Recommended)` / `Open as Ready`. A partial-feature PR is legitimate but should default to Draft so reviewers know it's mid-flight.

8. **Create the PR.** Use a HEREDOC for the body so the multi-line markdown is preserved verbatim. Add `--draft` only if the user picked Draft:

   ```bash
   gh pr create --title "<title>" --draft --body "$(cat <<'EOF'
   ## Summary
   - <bullet 1>
   - <bullet 2>

   ## Test plan
   - [ ] <check 1>
   - [ ] <check 2>
   EOF
   )"
   ```

   Do not pass `--base`, `--head`, `--repo`, `--reviewer`, `--assignee`, `--label`, or `--milestone` — let `gh` use the repo defaults. Changing those is a different intent (see Constraints). If `gh pr create` fails (auth scope, branch protection, repo policy, hook), surface its output verbatim and stop. Do not retry with different flags.

9. **Report.**
   - Print `gh pr view --json number,url,isDraft -q '"#\(.number) \(if .isDraft then "(draft) " else "" end)\(.url)"'`.
   - That single line is the report. Do not paste the body back into the conversation — the user can click the URL.

## PR title and body format

**Title** — one line, ≤ ~70 chars, imperative, lower-case, no trailing period. Reflect the *why* at a feature level, not the file list. With a spec present, prefer the spec `# Title` verbatim if it fits the length budget; otherwise paraphrase. Mirror any ticket-ID or scope-tag convention you detected in step 5.

**Body** — always two sections in this order:

```
## Summary
- <2–4 bullets at a feature level, not file-by-file>

## Test plan
- [ ] <verification step>
- [ ] <verification step>
```

- **Spec present** → Summary is 2–4 feature-level bullets distilled from `requirements.md` `## Background` (and, if it sharpens the picture, `plan.md`'s approach). Frame them as "what this feature does / why it exists", not "what was implemented in which file". Do **not** enumerate task titles — a feature is the unit, and a task list reads as a transcript. Test plan bullets come from the acceptance criteria in `requirements.md`: one checkbox per criterion, mirroring short ones verbatim and paraphrasing long ones into a single verifiable line. If the spec has unchecked tasks, add one trailing bullet to the Summary like `Note: M of N implementation tasks still open — see tasks.md` so reviewers aren't surprised.
- **No spec** → Summary is 2–4 feature-level bullets inferred from `git log <base>..HEAD` — describe what the branch *delivers*, not a per-commit log. Test plan bullets are a short generic checklist matched to the change shape — UI change → "click through the new screen"; API change → "exercise the endpoint locally"; refactor → "existing tests still pass"; tooling/config → "run the affected workflow once end-to-end".
- **No trailer.** Do not append `🤖 Generated with …`, `Co-Authored-By: Claude …`, or any other AI-attribution line. The PR description stands on its own.

## Constraints

These guardrails exist because the skill creates a publicly visible artifact on a shared remote — surprises here are costly to undo. Keep them tight:

- Stay in the main thread. Do not invoke the `Agent` tool — the commit hand-off (step 2) and the Draft/Ready prompt (step 7) are inherently interactive and a subagent can't carry them.
- Never force-push. No `--force`, no `--force-with-lease`, no `--no-verify` on `git push`. If a regular push is rejected, surface git's output and stop; the user decides how to recover (rebase, merge, change base).
- Never pass `--base`, `--head`, `--reviewer`, `--assignee`, `--label`, or `--milestone` to `gh pr create`. Let GitHub use the repo default base. If the user wants a different base, reviewers, or labels, that's a separate intent — bow out and let them ask explicitly so they get to choose deliberately.
- Do not edit source files. PR body synthesis is text-only. Code changes belong to `implement` / `review`; commits to `commit`.
- Treat `.vibepilot/spec/` as read-only. `create-pr` consumes the spec; editing it belongs to other skills.
- Do not paste the full diff, full commit log, or full PR body into the conversation. A one-line title preview in step 6 and the `gh pr view` URL in step 9 are the entire report. The user already has the diff locally and the rendered body on github.com.
- No AI-attribution trailers anywhere in the title or body. This is an explicit user preference for this codebase — `🤖 Generated with …` and `Co-Authored-By: Claude …` are forbidden in this skill, even if the underlying `commit` invocation would have added them elsewhere.
- If the auto-commit path is taken (step 2) and the user opts not to push from inside `commit`, do not "fix it up" by pushing yourself later in step 4 unless the branch is genuinely ahead at re-check time. The user's commit-time opt-out is authoritative for that commit; step 4 covers PR-time push only.
- On any `gh` failure (auth, rate limit, repo policy, branch protection), surface the output verbatim and stop. Do not retry with different flags — the failure mode almost always means the user needs to handle something out-of-band (grant a scope, talk to an admin, change branch protection).
