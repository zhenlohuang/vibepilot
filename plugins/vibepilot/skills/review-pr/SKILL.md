---
name: review-pr
description: Run a code-review pass over a GitHub pull request by delegating to the built-in `/review` skill (which reads the PR's diff and grades the changes). The argument selects which PR — empty defaults to the current branch's open PR, a number like `123` targets PR #123 in this repo, a URL targets that PR. Trigger whenever the user wants an existing GitHub PR reviewed — `/vibepilot:review-pr`, "review this PR", "review the pull request", "audit PR #123", "look at the PR", "code review on the PR", "do a pass on my PR", "审一下这个PR", "看看这个PR", "评审一下这个PR", "审查PR", "PR review一下", and similar — including when the user references the PR by number, URL, or just "the PR". Especially valid right after `/vibepilot:create-pr` opens a PR and the user wants a sanity pass, or when reviewing someone else's PR by URL. Do NOT trigger for auditing the *local* spec implementation against `.vibepilot/spec/` before a PR is opened — that's `/vibepilot:review`, a separate sibling skill (use it for phrases like "review my implementation", "check my work", "did I get the spec right", "审一下我的实现"). Do NOT trigger for opening a new PR (`/vibepilot:create-pr`), commenting on a PR, posting inline review comments, merging or approving a PR, requesting reviewers, or switching draft/ready state — those are different intents.
---

# vibepilot:review-pr — review a GitHub pull request

Run a code-review pass over an existing GitHub pull request and surface the result. The actual work — pulling the PR diff, reading the changed files, grading the change — is delegated to the **built-in `/review` skill** (a Claude Code built-in whose job is exactly that). This wrapper exists so the vibepilot command set has a discoverable `/vibepilot:`-namespaced entry point for PR review, clearly distinct from `/vibepilot:review`, which audits *local* spec implementation against `.vibepilot/spec/` before a PR is opened.

The skill is intentionally thin. The two reasons it exists at all rather than asking the user to type `/review` directly: (1) a consistent namespace next to `/vibepilot:create-pr` and `/vibepilot:commit` for the post-implementation workflow, and (2) an explicit boundary with the sibling `/vibepilot:review` so the auto-trigger doesn't conflate "review my local implementation" with "review this PR on GitHub".

## Preconditions

Check these first. If any fails, surface the state and stop — don't try to recover automatically, because "review a PR" stops being well-defined once the target is ambiguous:

- `gh` must be installed and authenticated (`gh auth status` succeeds). On failure, surface `gh auth login` as the fix and stop. PR review against GitHub requires `gh`; this skill cannot substitute a local-only path.
- A PR must be resolvable from the argument or the current branch.
  - With an empty argument, run `gh pr view --json number,url,title,state` against the current branch. If no PR exists, tell the user to either pass a PR number/URL explicitly or open one with `/vibepilot:create-pr` first — do not silently fall back to reviewing local diff (that's `/vibepilot:review`'s job).
  - With a number or URL, run `gh pr view <arg> --json number,url,title,state` to confirm it exists. If the PR is `MERGED` or `CLOSED`, surface the state once and ask whether to proceed — historical review is legitimate but the user should opt in.

## Argument parsing

The argument selects the PR:

- Empty → current branch's PR (detected via `gh pr view`)
- A number `123` → PR #123 in the current repo
- A URL `https://github.com/owner/repo/pull/123` → that PR (across repos is fine; `gh` handles it)

If the input is unparseable (e.g., `123abc`, a branch name), ask the user once to clarify which PR they mean — don't guess.

## Steps

1. **Resolve the PR.** Apply argument parsing + preconditions. Once resolved, print one confirmation line so the user can see what's being reviewed:

   ```
   Reviewing PR #<number> — <title>
   ```

   Just this one line. Do not paste the PR body, the diff, the commit list, or the reviewer roster — the built-in `/review` skill will surface what it needs.

2. **Delegate to the built-in `/review` skill** via the `Skill` tool with `skill: "review"`. Pass the resolved PR number or URL as `args` so the built-in skill has an unambiguous target rather than re-deriving it from the working tree. The built-in skill owns the diff fetch, file reads, and grading — do not duplicate any of that work in this thread.

3. **Relay the result.** When the built-in skill returns, surface its summary and verdict to the user verbatim, in the language it returned. Do not paraphrase, do not re-grade, and do not paste the full diff back into the conversation. The PR URL plus the built-in skill's output is the whole report.

## Language

Match the user's conversation language for the surface text you write yourself — the "Reviewing PR …" confirmation line, any precondition-failure message, and any framing around the relayed result. Keep verdict tokens (`PASS`, `ADVISORY`, `BLOCK`), command names (`gh`, `/vibepilot:review`, `/vibepilot:create-pr`), and conventional identifiers in canonical English so downstream tooling and shared muscle memory stay deterministic.

## Constraints

These guardrails exist because the whole point of the skill is to be a *thin* boundary — every behavior it accretes is a behavior the built-in `/review` skill might do differently, and the two will diverge silently. Keep them tight:

- **Thin wrapper, period.** Do not read the PR diff, open source files, or write a review report yourself in the main thread. That is the built-in `/review` skill's job — bypassing it defeats the entire reason for delegation.
- **Sibling-skill boundary.** Do not run this skill when the user wants to audit local spec implementation against `.vibepilot/spec/` before a PR exists. That intent belongs to `/vibepilot:review`. If the user invoked this skill but the context strongly suggests they meant the local audit (no PR exists for the current branch, `.vibepilot/spec/tasks.md` has unchecked items, and the phrasing is about "my implementation" rather than "the PR"), redirect to `/vibepilot:review` and stop.
- **Do not modify code, edit the PR title or body, or post comments** on the user's behalf. Review output is conversational; turning it into a PR comment is a separate explicit intent.
- **Do not create, merge, approve, request reviewers on, or change the draft/ready state of a PR.** Those are different intents — bow out so the user gets to ask for them deliberately.
- **Treat `.vibepilot/spec/` as read-only and, in this skill, untouched.** The built-in `/review` skill works off the PR diff; do not "supplement" by reading the local spec into this thread.
- **One precondition surface, then stop.** On `gh auth` failure or an unresolvable PR target, surface the failure verbatim and stop. Do not retry, do not guess a different PR, do not silently proceed against an inferred target — a wrong target on a public PR is costly to undo.
