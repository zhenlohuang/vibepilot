---
name: commit
description: Create a Conventional Commits git commit for the current working tree. Uses the active `.vibepilot/spec/` for message context when present; falls back to a diff-derived message otherwise. Trigger when the user runs `/vibepilot:commit`, asks to commit / save / land the changes, or says something like "commit this", "make a commit", or "let's commit" — typically after `/vibepilot:review` reports PASS, but also valid standalone. Do NOT trigger for amending, rewriting history, force-pushing, or for pushing alone without a fresh commit.
---

# vibepilot:commit — create a conventional commit

You are creating a single git commit from the current working tree using the Conventional Commits format. This work happens in the main thread on purpose — staging decisions and the post-commit push prompt both need direct back-and-forth with the user, and the diff for one feature's worth of changes is bounded enough that reading it inline doesn't pollute the thread.

The skill is **generic**: it works with or without an active `.vibepilot/spec/`. When a spec is present, the commit message body draws from `requirements.md` and the `- [x]` lines in `tasks.md`. When no spec exists, the body is synthesized from the staged diff.

## Preconditions

- A git repository must exist at or above the working directory (`git rev-parse --git-dir` succeeds). If not, tell the user this isn't a git repo and stop.
- The working tree must have something to commit (staged changes, modified tracked files, or untracked files). If everything is already clean and committed, output `Working tree clean — nothing to commit.` and stop.
- The repository must not be mid-rebase, mid-merge, or in detached HEAD. If `git status` reveals any of those, surface the state and stop — don't try to recover from it.

## Language

Match the language the user is using in this conversation (and, if a spec is active, the language of `requirements.md`) for the **subject** and **body** of the commit message. The Conventional Commits type token (`feat`, `fix`, `refactor`, `docs`, `test`, `chore`, …), the optional scope, section structure, and the trailer stay in canonical English so tooling and downstream readers can parse them deterministically.

## Steps

1. **Gather working-tree state.** Run these in parallel and inspect the output:
   - `git rev-parse --git-dir` — confirm we're in a repo
   - `git status --short` — what's modified / staged / untracked
   - `git diff --stat` — unstaged change shape
   - `git diff --cached --stat` — staged change shape
   - `git log --oneline -10` — recent message style to match
   - `git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD` — current branch (or detached HEAD)

   Apply the preconditions above. If a rebase/merge is in progress (`.git/rebase-*` or `.git/MERGE_HEAD` referenced in status), stop and tell the user to resolve it first.

2. **Decide what to stage.**
   - **If anything is already staged** → respect the user's staging. Commit only what's in the index; leave unstaged work alone.
   - **Else if there are unstaged modifications to tracked files** → run `git add -u` (tracked files only). Never `git add -A` or `git add .` — those can sweep in secrets (`.env`, credentials) or large binaries the user didn't intend to commit.
   - **Untracked files present?** Ask via AskUserQuestion with the file list shown:
     - `Skip untracked (Recommended)` — commit only tracked changes
     - `Include specific untracked files` — user names which to add; stage by exact path
   - If after staging there is still nothing in the index, stop with `Nothing staged to commit.`

3. **Read context for the message.**
   - If `.vibepilot/spec/requirements.md` exists with substantive content: read its `# Title`, the `## Background` paragraph, and the full `tasks.md`. Extract every `- [x]` line — those become the body bullets.
   - Otherwise: read `git diff --cached` (cap at roughly the first ~400 lines if huge) to infer scope.
   - Either way: scan the `git log --oneline -10` output. If the repo's recent commits clearly use a non-standard style (e.g., emoji prefixes, ticket-ID prefixes, or a different convention), keep Conventional Commits as the default but mirror the local convention's accent (e.g., include a ticket ID if every recent commit has one).

4. **Synthesize the commit message** using the format below. Do **not** ask the user to approve the message before committing — they chose this skill to delegate that. Surface a one-line preview in the report instead.

5. **Commit.** Use a HEREDOC so the multi-line body is preserved verbatim:

   ```bash
   git commit -m "$(cat <<'EOF'
   <type>(<scope>): <subject>

   <body>
   EOF
   )"
   ```

   No `--amend`, no `--no-verify`, no `--no-gpg-sign`. If a pre-commit hook fails, surface the hook's output to the user and stop — do **not** retry with `--no-verify`, and do **not** `--amend` a previous commit to recover. The user fixes the underlying issue and re-runs `/vibepilot:commit`.

6. **Report and ask about push.**
   - Print `git log -1 --oneline` so the new commit is visible.
   - Ask once via AskUserQuestion:
     - `Don't push (Recommended)` — leave the commit local
     - `Push to remote` — run `git push` (no flags; no `--force`, no `-u`)
   - If the user picks push and `git push` fails because no upstream is set, surface git's hint (`git push --set-upstream origin <branch>`) and stop — don't decide the upstream for the user.

## Commit message format

```
<type>(<scope>): <subject under ~70 chars>

<body>
```

- **type** — infer from spec keywords (or diff if no spec):
  - new file / endpoint / command / capability → `feat`
  - bug language ("fix", regression, broken) → `fix`
  - rename / restructure with no behavior change → `refactor`
  - docs-only changes → `docs`
  - test-only changes → `test`
  - tooling / config / build / CI → `chore`
- **scope** (optional) — the top-level directory of the change (`plugins`, `agents`, `skills/commit`, …) or the spec slug if there's an obvious one. Omit when the change spans the whole repo.
- **subject** — imperative, lower-case, no trailing period. Reflect the *why* at a feature level, not the file list.
- **body** —
  - **Spec present** → bullet the completed task titles from `tasks.md` (the text after `- [x] N.`), in order.
  - **No spec** → 2–4 bullets summarizing the diff at a feature level, not file-by-file.
- **No trailer.** Do **not** append `🤖 Generated with …` or `Co-Authored-By` lines. The commit stands on its own message.

## Constraints

- Do **not** invoke the `Agent` tool — this work belongs in the main thread for the staging and push prompts.
- Do **not** use `git add -A` or `git add .` — always `git add -u` or explicit paths.
- Do **not** use `--amend`, `--no-verify`, `--no-gpg-sign`, `--force`, or any history-rewriting flag.
- Do **not** push without an explicit affirmative answer to the post-commit prompt.
- Do **not** modify any file under `.vibepilot/spec/` — `commit` is strictly read-only against the spec.
- Do **not** edit source files — message synthesis is text-only.
- Do **not** retry a failed commit by bypassing the pre-commit hook. Surface the failure and stop.
- Do **not** paste the full diff or full commit body into the conversation — a one-line preview plus `git log -1 --oneline` is enough.
