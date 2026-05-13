---
name: create-pr
description: Open a pull request for the current branch.
disable-model-invocation: true
---

# /vibepilot:create-pr

Granularity is **one PR per feature**. With a spec active, `requirements.md` (description + acceptance criteria) and `plan.md` (approach) drive title and body; `tasks.md` is consulted only for completion state, **never enumerated** into the body. Without a spec, title and body synthesize from `git log <base>..HEAD`.

## Preconditions

Surface state and stop if any fails:

- Git repo (`git rev-parse --git-dir` succeeds).
- `gh` installed and authenticated (`gh auth status`). On failure, surface `gh auth login` as the fix.
- Not mid-rebase, mid-merge, mid-cherry-pick, or in detached HEAD.
- Current branch is **not** the repo's default branch (detect via `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`).
- No open PR exists for this branch (`gh pr list --head <branch> --state open --json url,number,isDraft`). If one does, surface its URL and stop.

## Steps

1. **Gather state** (parallel):
   - `git rev-parse --git-dir`, `gh auth status`
   - `git status --short` — anything to commit?
   - `git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD`
   - `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
   - `git status --porcelain=v2 --branch` — upstream / ahead-behind

   Then in parallel:
   - `gh pr list --head <branch> --state open --json url,number,isDraft`
   - `git rev-list --left-right --count <base>...HEAD`
   - `git log <base>..HEAD --oneline`

2. **Handle uncommitted changes by delegating to `commit`.** If `git status --short` is non-empty, invoke `/vibepilot:commit` (or, if your harness can't, read `plugins/vibepilot/skills/commit/SKILL.md` and follow its Steps inline). After it completes, re-run step 1. If the user aborts and the tree is still dirty, stop.

3. **Surface "behind base" as a warning, not a block.** Print one line: `Branch is N commits behind <base>. PR will still open; rebase or merge later if you want a linear history.` and continue.

4. **Push the branch if needed.**
   - No upstream → `git push -u origin <branch>`.
   - Upstream + ahead → `git push` (no flags).
   - Up-to-date → skip.
   - On push failure, surface git's output verbatim and stop. **No `--force` / `--force-with-lease`.**

5. **Read context for title and body.**
   - **Spec present** → this PR represents the whole feature. Read:
     - `# Title` → strong title candidate
     - `## Background` paragraph → primary input for Summary (the *why* and *what*, user-facing)
     - Acceptance Criteria → primary input for Test plan
     - `plan.md` (high-level approach) → flavor for Summary, not its skeleton
     - `tasks.md` → **only** for completion state. Count `- [ ]` vs `- [x]`. **Do not enumerate task titles into the body** — task lists read as transcripts; PR readers want feature behavior. If some `- [ ]` remain, append a `Note: M of N implementation tasks still open` line in Summary.
     - `.vibepilot/spec/review.md` if present → `PASS` outcome biases step 7 toward Ready; anything else toward Draft.
   - **No spec** → infer feature shape from `git log <base>..HEAD --oneline` (already gathered). If thin, supplement once with `git diff <base>..HEAD --stat`. Don't read the full diff.
   - **In both cases** → if recent merged PRs show a clear convention (ticket-ID prefix `[ABC-123]`, scope tag `feat:`), mirror it. Run `gh pr list --state merged --limit 10 --json title -q '.[].title'` once if unsure. Don't invent a convention.

6. **Synthesize title and body** per Format. Don't ask for approval — surface a one-line preview in the report.

7. **Ask Draft vs Ready** via AskUserQuestion:
   - `review.md` PASS **and** all tasks `- [x]` (or no spec + small self-contained change) → `Open as Ready (Recommended)` / `Open as Draft`.
   - Otherwise (no review, non-PASS, any unchecked task, or substantial scope without review) → `Open as Draft (Recommended)` / `Open as Ready`.

8. **Create the PR** with HEREDOC for the body. Add `--draft` only if user picked Draft:

   ```bash
   gh pr create --title "<title>" --draft --body "$(cat <<'EOF'
   ## Summary
   - <bullet>

   ## Test plan
   - [ ] <check>
   EOF
   )"
   ```

   Don't pass `--base`, `--head`, `--repo`, `--reviewer`, `--assignee`, `--label`, or `--milestone`. On `gh pr create` failure, surface output verbatim and stop.

9. **Report.** Print `gh pr view --json number,url,isDraft -q '"#\(.number) \(if .isDraft then "(draft) " else "" end)\(.url)"'`. That single line is the report.

## Format

**Title** — one line, ≤ ~70 chars, imperative, lower-case, no trailing period. The *why* at a feature level. With spec: prefer `# Title` verbatim if it fits; else paraphrase. Mirror detected ticket/scope conventions.

**Body** — always two sections, in this order:

```
## Summary
- <2–4 feature-level bullets>

## Test plan
- [ ] <verification step>
```

- **Spec present** → Summary distills `requirements.md` `## Background` (and `plan.md`'s approach if it sharpens the picture). Frame as "what the feature does / why", not "what was implemented in which file". **Do not enumerate task titles.** Test plan = one checkbox per Acceptance Criterion, mirroring short ones verbatim and paraphrasing long ones.
- **No spec** → Summary = 2–4 feature-level bullets inferred from `git log <base>..HEAD`. Test plan = short generic checklist matched to change shape (UI → "click through new screen"; API → "exercise endpoint locally"; refactor → "existing tests pass"; tooling → "run affected workflow once").

## Constraints

- Match the user's conversation language for title and body (and `requirements.md`'s language with a spec). Section headers `## Summary` / `## Test plan` and any conventional tokens stay canonical English.
- Stay in the main thread — do not invoke the `Agent` tool.
- Never force-push. No `--force`, no `--force-with-lease`, no `--no-verify` on `git push`. On rejection, surface and stop; the user decides recovery.
- Never pass `--base`, `--head`, `--reviewer`, `--assignee`, `--label`, or `--milestone` to `gh pr create`. Those are different intents — let the user ask deliberately.
- Do not edit source files. Treat `.vibepilot/spec/` as read-only.
- Do not paste full diff, full commit log, or full PR body — title preview + `gh pr view` URL is the whole report.
- **No AI-attribution trailers** anywhere in the title or body.
- If the auto-commit path (step 2) was taken and the user opted not to push from inside `commit`, do not push later in step 4 unless the branch is genuinely ahead at re-check.
- On any `gh` failure (auth, rate limit, policy, branch protection), surface verbatim and stop. Don't retry with different flags.
