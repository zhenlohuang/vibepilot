---
name: commit
description: Create a single Conventional Commits git commit from the current working tree, then ask whether to push. Pulls message context from the active `.vibepilot/spec/` (requirements + completed task titles) when present; otherwise synthesizes from the staged diff. Trigger whenever the user wants a commit made for them — `/vibepilot:commit`, "commit this", "commit the changes", "make a commit", "save my progress", "ship it", "let's land it", "check this in", "提交一下", "提交代码", and similar — including standalone use without an active spec. Especially valid after `/vibepilot:review` reports PASS or `/vibepilot:implement` finishes a task. Do NOT trigger for amending or rewording an existing commit, rewriting history (rebase, squash, fixup), force-pushing, pushing without making a new commit, or staging files without committing.
---

# vibepilot:commit — create a conventional commit

Create one Conventional Commits commit from the current working tree, then offer to push. This runs in the main thread because staging decisions and the push prompt both need user back-and-forth, and a single feature's worth of diff is bounded enough that reading it inline doesn't pollute the thread.

The skill is **generic**: it works with or without an active `.vibepilot/spec/`. With a spec, the body bullets come from completed (`- [x]`) tasks. Without one, the body is synthesized from the staged diff.

## Preconditions

Check these first. If any fails, surface the state and stop — don't try to recover automatically, because "create a commit" stops being well-defined once the repo is mid-operation:

- A git repository must exist (`git rev-parse --git-dir` succeeds). If not, tell the user this isn't a git repo and stop.
- Something must be committable — staged, modified-tracked, or untracked. If the working tree is fully clean, output `Working tree clean — nothing to commit.` and stop.
- The repo must not be mid-rebase, mid-merge, mid-cherry-pick, or in detached HEAD. The user should resolve that state first; this skill won't attempt it.

## Language

Match the user's conversation language for the **subject** and **body** of the message (and, when a spec is active, the language of `requirements.md`). Keep the Conventional Commits type token (`feat`, `fix`, `refactor`, `docs`, `test`, `chore`), the optional scope, the section structure, and any trailers in canonical English so tooling and downstream readers can parse them deterministically.

## Steps

1. **Gather working-tree state.** Run these in parallel and inspect:
   - `git rev-parse --git-dir` — repo check
   - `git status --short` — modified / staged / untracked
   - `git diff --stat` and `git diff --cached --stat` — change shape
   - `git log --oneline -10` — to match the repo's recent message style
   - `git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD` — detect detached HEAD

   Apply the preconditions. If `.git/rebase-*`, `.git/MERGE_HEAD`, or `.git/CHERRY_PICK_HEAD` is referenced in status, stop and ask the user to resolve.

2. **Decide what to stage.**
   - **Anything already staged** → respect the user's staging. Commit only what's in the index; leave the unstaged work alone.
   - **Otherwise, tracked files modified** → run `git add -u` (tracked files only). Never `git add -A` or `git add .` — those can sweep in secrets (`.env`, credentials) or large binaries the user didn't intend to commit, and that mistake is expensive to undo.
   - **Untracked files present** → ask via AskUserQuestion with the file list shown:
     - `Skip untracked (Recommended)` — commit only tracked changes
     - `Include specific untracked files` — user names which to add; stage by exact path
   - If after staging the index is still empty, stop with `Nothing staged to commit.`

3. **Read context for the message.**
   - **If `.vibepilot/spec/requirements.md` exists with substantive content** → read its `# Title`, the `## Background` paragraph, and `tasks.md`. The completed (`- [x]`) task titles become the body bullets, in order. If there are more than ~6 completed tasks, group them into 3–5 thematic bullets rather than dumping the raw list — the goal is a readable message, not a transcript.
   - **Otherwise** → read `git diff --cached` (cap at roughly the first ~400 lines if huge) to infer scope at a feature level.
   - **In both cases** → scan `git log --oneline -10`. If the repo clearly uses a local convention — ticket-ID prefixes, emoji headers, `Signed-off-by` trailers on every commit — mirror that accent. Conventional Commits stays the default backbone.

4. **Synthesize the commit message** using the format below. Do **not** ask the user to approve the message before committing — they chose this skill to delegate that. Surface a one-line preview in the report instead.

5. **Commit.** Use a HEREDOC so the multi-line body is preserved verbatim:

   ```bash
   git commit -m "$(cat <<'EOF'
   <type>(<scope>): <subject>

   <body>
   EOF
   )"
   ```

   No `--amend`, `--no-verify`, `--no-gpg-sign`, `--allow-empty`, or `--force`. Do not pass `-s` / `--signoff` unless every recent commit in the log carries `Signed-off-by:` (e.g., a DCO-required repo) — in that case mirror it. If a pre-commit hook fails, surface the first ~30 lines of the hook output so the user can see the failure, then stop. Do not retry with `--no-verify`, and do not `--amend` a previous commit to recover — the user fixes the underlying issue and re-runs `/vibepilot:commit`.

6. **Report and ask about push.**
   - Print `git log -1 --oneline` so the new commit is visible.
   - Ask once via AskUserQuestion:
     - `Don't push (Recommended)` — leave the commit local
     - `Push to remote` — run `git push` (no flags; no `--force`, no `-u`)
   - If `git push` fails because no upstream is set, surface git's own hint (`git push --set-upstream origin <branch>`) and stop — don't decide the upstream for the user. For other push failures (rejected, fetch-first, hooks), surface git's output verbatim and stop.

## Commit message format

```
<type>(<scope>): <subject under ~70 chars>

<body>
```

- **type** — infer from spec keywords (or diff when there's no spec):
  - new file / endpoint / command / capability → `feat`
  - bug language ("fix", regression, broken) → `fix`
  - rename / restructure with no behavior change → `refactor`
  - docs-only changes → `docs`
  - test-only changes → `test`
  - tooling / config / build / CI → `chore`
- **scope** (optional) — the top-level directory of the change (`plugins`, `agents`, `skills/commit`, …) or the spec slug if there's an obvious one. Omit when the change spans the whole repo.
- **subject** — imperative, lower-case, no trailing period. Reflect the *why* at a feature level, not the file list.
- **body** —
  - **Spec present** → bullet the completed task titles (text after `- [x] N.`), in order; condense if there are many.
  - **No spec** → 2–4 bullets summarizing the diff at a feature level, not file-by-file.
- **No trailer.** Do not append `🤖 Generated with …` or `Co-Authored-By` lines. The commit stands on its own message.

## Constraints

These guardrails exist because the skill sits at a sensitive boundary — wrong actions here can leak secrets, lose work, or surprise collaborators. Keep them tight:

- Stay in the main thread. Do not invoke the `Agent` tool — staging choices and the push prompt are inherently interactive and a subagent can't carry them.
- Stage with `git add -u` or explicit paths only. `git add -A` / `git add .` is how `.env` files and `node_modules` end up in commits; an avoidable disaster.
- No history-rewriting or safety-bypass flags: `--amend`, `--no-verify`, `--no-gpg-sign`, `--force`, `--allow-empty`. If the user wants to amend or rewrite, that's a different intent — bow out and let them ask for it explicitly.
- Push only after an explicit affirmative answer to the post-commit prompt. A silent push surprises the user and is hard to undo on a shared branch.
- Treat `.vibepilot/spec/` as read-only. `commit` consumes the spec; editing the spec belongs to other skills.
- Do not edit source files — message synthesis is text-only. Code changes belong in `implement` / `review`.
- On hook failure, surface the output and stop. Bypassing a hook silently defeats whatever check the user (or their team) set up, so the cost of a "helpful" retry is much higher than the cost of asking them to fix it.
- Do not paste the full diff or full commit body into the conversation. The user already has the diff in their working tree; a one-line preview plus `git log -1 --oneline` is the whole report.
