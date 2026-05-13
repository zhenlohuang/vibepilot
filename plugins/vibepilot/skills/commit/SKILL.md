---
name: commit
description: Commit the current changes and offer to push.
disable-model-invocation: true
---

# /vibepilot:commit

Generic: works with or without an active `.vibepilot/spec/`. With a spec, body bullets come from completed (`- [x]`) task titles; without one, from the staged diff.

## Preconditions

Surface state and stop if any fails — recovery isn't well-defined:

- Git repo (`git rev-parse --git-dir` succeeds).
- Something committable (staged / modified-tracked / untracked). If clean: `Working tree clean — nothing to commit.` and stop.
- Not mid-rebase, mid-merge, mid-cherry-pick, or in detached HEAD.

## Steps

1. **Gather state** (parallel):
   - `git rev-parse --git-dir`
   - `git status --short`
   - `git diff --stat` and `git diff --cached --stat`
   - `git log --oneline -10` — to match the repo's recent message style
   - `git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD` — detect detached HEAD

   Apply preconditions. If `.git/rebase-*`, `.git/MERGE_HEAD`, or `.git/CHERRY_PICK_HEAD` is referenced, stop and ask the user to resolve.

2. **Decide what to stage.**
   - **Already staged** → respect the index; commit only what's staged.
   - **Tracked files modified, nothing staged** → `git add -u` (tracked only). Never `git add -A` / `git add .`.
   - **Untracked files present** → ask via AskUserQuestion with the file list shown:
     - `Skip untracked (Recommended)`
     - `Include specific untracked files` (user names which; stage by exact path)
   - If after staging the index is still empty, stop with `Nothing staged to commit.`

3. **Read context for the message.**
   - **Spec present (substantive `requirements.md`)** → read its `# Title`, `## Background`, and `tasks.md`. Completed (`- [x]`) task titles become body bullets, in order. If more than ~6 completed tasks, condense into 3–5 thematic bullets — readable, not a transcript.
   - **No spec** → read `git diff --cached` (cap ~400 lines) to infer scope at a feature level.
   - **In both cases** → scan `git log --oneline -10`. Mirror local conventions if obvious (ticket-ID prefix, emoji headers, `Signed-off-by` on every commit). Conventional Commits stays the backbone.

4. **Synthesize the message** using the format below. Do not ask the user to approve before committing — they delegated that. Surface a one-line preview in the report.

5. **Commit** with HEREDOC for the body:

   ```bash
   git commit -m "$(cat <<'EOF'
   <type>(<scope>): <subject>

   <body>
   EOF
   )"
   ```

   Don't pass `-s` / `--signoff` unless every recent commit has `Signed-off-by:` (DCO repo). On pre-commit hook failure, surface the first ~30 lines and stop — don't retry with `--no-verify`, don't `--amend` to recover.

6. **Report and ask about push.**
   - Print `git log -1 --oneline`.
   - Ask via AskUserQuestion: `Don't push (Recommended)` / `Push to remote` (`git push`, no flags; no `--force`, no `-u`).
   - If push fails because no upstream, surface git's hint (`git push --set-upstream origin <branch>`) and stop — don't decide the upstream.

## Format

```
<type>(<scope>): <subject under ~70 chars>

<body>
```

- **type** — `feat` (new file / endpoint / command / capability), `fix` (bug / regression), `refactor` (no behavior change), `docs`, `test`, `chore` (tooling / config / CI).
- **scope** (optional) — top-level dir (`plugins`, `skills/commit`, …) or spec slug. Omit when change spans the whole repo.
- **subject** — imperative, lower-case, no trailing period. The *why* at a feature level, not the file list.
- **body** — spec present → completed task titles in order, condensed if many. No spec → 2–4 feature-level bullets summarizing the diff.

## Constraints

- Match the user's conversation language for subject and body (and `requirements.md`'s language when a spec is active). Conventional Commits type tokens, scope, and any trailers stay canonical English so tooling can parse them.
- Stay in the main thread — do not invoke the `Agent` tool.
- Stage with `git add -u` or explicit paths only. Never `-A` / `.`.
- No `--amend`, `--no-verify`, `--no-gpg-sign`, `--force`, `--allow-empty`.
- Push only after explicit affirmative answer.
- Treat `.vibepilot/spec/` as read-only. No source-file edits.
- **No AI-attribution trailers.** Do not append `🤖 Generated with …` or `Co-Authored-By: Claude …`.
- Do not paste full diff or full commit body — one-line preview + `git log -1 --oneline` is the whole report.
