---
description: Commit the current changes and offer to push.
---

# /vibepilot:commit

Works with or without an active `.vibepilot/work/`. With a spec, body bullets come from completed (`- [x]`) task titles; without one, from the staged diff. Stays in the main thread — no `Agent` calls.

## Preconditions

Surface and stop on any failure — recovery isn't well-defined here:

- Git repo (`git rev-parse --git-dir`).
- Something committable. If clean: `Working tree clean — nothing to commit.` and stop.
- Not mid-rebase/merge/cherry-pick, not in detached HEAD.

## Flow

1. **Gather state** in parallel: `git rev-parse --git-dir`, `git status --short`, `git diff --stat`, `git diff --cached --stat`, `git log --oneline -10`, `git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD`. If `.git/rebase-*`, `MERGE_HEAD`, or `CHERRY_PICK_HEAD` exists, stop and ask the user to resolve.

2. **Decide what to stage.**
   - Already staged → respect the index.
   - Tracked-but-unstaged modifications, nothing staged → `git add -u` (tracked only). **Never `git add -A` / `git add .`.**
   - Untracked files present → AskUserQuestion with the file list: `Skip untracked (Recommended)` / `Include specific untracked files` (user names which; stage by exact path).
   - Index still empty after staging → `Nothing staged to commit.` and stop.

3. **Read context for the message.**
   - Spec present (substantive `requirements.md`) → read `# Title`, `## Background`, and `tasks.md`. Completed task titles become body bullets in order; if more than ~6, condense to 3–5 thematic bullets.
   - No spec → read `git diff --cached` (cap ~400 lines) to infer feature-level scope.
   - Always → scan `git log --oneline -10`. Mirror obvious local conventions (ticket-ID prefix, emoji headers, `Signed-off-by` on every commit). Conventional Commits stays the backbone.

4. **Synthesize** per Format below. Don't ask the user to approve — they delegated that. Surface a one-line preview in the report.

5. **Commit** with HEREDOC for the body. Don't pass `-s`/`--signoff` unless every recent commit has `Signed-off-by:` (DCO repo). On pre-commit hook failure, surface the first ~30 lines and stop — no `--no-verify`, no `--amend` to recover.

6. **Report and offer push.**
   - Print `git log -1 --oneline`.
   - AskUserQuestion: `Don't push (Recommended)` / `Push to remote` (`git push`, no flags).
   - If push fails because no upstream, surface git's hint (`git push --set-upstream origin <branch>`) and stop — don't pick the upstream.

## Format

```
<type>(<scope>): <subject>

<body>
```

- **type** — `feat` (new capability), `fix`, `refactor` (no behavior change), `docs`, `test`, `chore` (tooling/config/CI).
- **scope** (optional) — top-level dir (`plugins`, `commands/commit`, …) or spec slug. Omit for repo-wide changes.
- **subject** — imperative, lower-case, no trailing period, ≤ ~70 chars. The *why* at a feature level, not a file list.
- **body** — spec present → completed task titles in order, condensed if many. No spec → 2–4 feature-level bullets from the diff.

## Notes

- Subject and body match the user's conversation language (and `requirements.md`'s language when a spec is active). Conventional Commits tokens, scope, and trailers stay English.
- Stage with `git add -u` or explicit paths only. Never `-A` / `.`.
- No `--amend`, `--no-verify`, `--no-gpg-sign`, `--force`, `--allow-empty`.
- Push only after an affirmative answer.
- Treat `.vibepilot/work/` as read-only. No source-file edits.
- **No AI-attribution trailers** (`🤖 Generated …`, `Co-Authored-By: Claude …`).
- One-line preview + `git log -1 --oneline` is the whole report — no full diff or full commit body.
