---
description: Open a pull request for the current branch.
---

# /vibepilot:create-pr

Granularity is **one PR per feature**. With a spec, `requirements.md` (description + acceptance criteria) and `plan.md` (approach) drive title and body; `tasks.md` is consulted only for completion state, **never enumerated**. Without a spec, title and body synthesize from `git log <base>..HEAD`. Stays in the main thread — no `Agent` calls.

## Preconditions

Surface and stop on any failure:

- Git repo (`git rev-parse --git-dir`).
- `gh` installed and authenticated (`gh auth status`). On failure, surface `gh auth login`.
- Not mid-rebase/merge/cherry-pick, not in detached HEAD.
- Current branch is **not** the default branch (`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`).
- No open PR for this branch (`gh pr list --head <branch> --state open --json url,number,isDraft`). If one exists, surface its URL and stop.

## Flow

1. **Gather state** in parallel: `git rev-parse --git-dir`, `gh auth status`, `git status --short`, `git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD`, `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`, `git status --porcelain=v2 --branch`. Then: `gh pr list --head <branch> --state open --json url,number,isDraft`, `git rev-list --left-right --count <base>...HEAD`, `git log <base>..HEAD --oneline`.

2. **Uncommitted changes** → delegate to `/vibepilot:commit` (or, if the harness can't, read `commit.md` and follow it inline). Re-run step 1 after. If the user aborts and the tree is still dirty, stop.

3. **Behind base** is a warning, not a block: `Branch is N commits behind <base>. PR will still open; rebase or merge later if you want a linear history.` Continue.

4. **Push if needed.** No upstream → `git push -u origin <branch>`. Upstream + ahead → `git push`. Up-to-date → skip. On failure, surface git output verbatim and stop. **No `--force` / `--force-with-lease`.**

5. **Read context for title and body.**
   - Spec present → this PR is the whole feature. Read `# Title` (strong title candidate), `## Background` (primary input for Summary — the *why* and *what*, user-facing), Acceptance Criteria (primary input for Test plan), `plan.md` (flavor for Summary, not its skeleton). Consult `tasks.md` **only** for completion state — count `- [ ]` vs `- [x]`. **Do not enumerate task titles** — task lists read as transcripts; PR readers want feature behavior. If any `- [ ]` remain, append `Note: M of N implementation tasks still open` in Summary. Read `review.md` if present — `PASS` biases step 7 toward Ready.
   - No spec → infer feature shape from `git log <base>..HEAD --oneline`. If thin, supplement once with `git diff <base>..HEAD --stat`. Don't read the full diff.
   - Always → if recent merged PRs show a clear convention (ticket prefix `[ABC-123]`, scope tag `feat:`), mirror it. Run `gh pr list --state merged --limit 10 --json title -q '.[].title'` once if unsure. Don't invent a convention.

6. **Synthesize** per Format. Don't ask for approval — surface a one-line preview in the report.

7. **Ask Draft vs Ready** via AskUserQuestion:
   - `review.md` PASS **and** all tasks `- [x]` (or no spec + small self-contained change) → `Open as Ready (Recommended)` / `Open as Draft`.
   - Otherwise (no review, non-PASS, any unchecked task, or substantial scope without review) → `Open as Draft (Recommended)` / `Open as Ready`.

8. **Create the PR** with `gh pr create --title "<title>" --body "$(cat <<'EOF' … EOF)"`. Add `--draft` only if the user chose Draft. Don't pass `--base`, `--head`, `--repo`, `--reviewer`, `--assignee`, `--label`, `--milestone`. On failure, surface output verbatim and stop.

9. **Report.** Print `gh pr view --json number,url,isDraft -q '"#\(.number) \(if .isDraft then "(draft) " else "" end)\(.url)"'`. That single line is the report.

## Format

**Title** — one line, ≤ ~70 chars, imperative, lower-case, no trailing period, feature-level *why*. With spec: prefer `# Title` verbatim if it fits; else paraphrase. Mirror detected ticket/scope conventions.

**Body** — always two sections, in this order:

```
## Summary
- <2–4 feature-level bullets>

## Test plan
- [ ] <verification step>
```

- Spec present → Summary distills `requirements.md` `## Background` (and `plan.md`'s approach if it sharpens the picture). Frame as "what the feature does / why", not "what was implemented in which file". Test plan = one checkbox per Acceptance Criterion (mirror short ones verbatim, paraphrase long ones).
- No spec → Summary = 2–4 feature-level bullets inferred from `git log <base>..HEAD`. Test plan = short generic checklist matched to change shape (UI → click-through; API → exercise endpoint; refactor → existing tests pass; tooling → run affected workflow).

## Notes

- Title and body match the user's conversation language (and `requirements.md`'s language with a spec). `## Summary` / `## Test plan` and conventional tokens stay English.
- Never force-push. No `--force`, `--force-with-lease`, `--no-verify` on `git push`. On rejection, surface and stop.
- Don't pass `--base`, `--head`, `--reviewer`, `--assignee`, `--label`, `--milestone` to `gh pr create` — those are different intents.
- Treat `.vibepilot/work/` as read-only. Don't edit source files.
- **No AI-attribution trailers** in title or body.
- Title preview + `gh pr view` URL is the whole report — no full diff, log, or body.
- If step 2 took the auto-commit path and the user opted not to push from inside `commit`, don't push later in step 4 unless the branch is genuinely ahead at re-check.
- On any `gh` failure (auth, rate limit, policy, branch protection), surface verbatim and stop.
