---
name: implement
description: Build the code for one or more tasks of the current spec.
disable-model-invocation: true
argument-hint: "[N | N-M | N,M,K | all]"
---

# /vibepilot:implement

All coding work is delegated to `vibe-developer` — even small edits, for consistency.

## Preconditions

- `.vibepilot/spec/{requirements,plan,tasks}.md` must all have substantive content. If any is missing or a placeholder, tell the user which step to run first and stop.
- `tasks.md` must contain at least one `- [ ]`. If everything is `- [x]`, tell the user the implementation is complete and suggest `/vibepilot:review`.

## Argument parsing

- Empty → the next single unchecked task (lowest-numbered `- [ ]`)
- `3` → just task 3
- `1-3` → tasks 1 through 3 inclusive (reversed range like `3-1` is invalid — ask to clarify)
- `1,3,5` → exactly those tasks
- `all` → every remaining unchecked task, in order

Validate against actual task numbers in `tasks.md` before delegating:
- Missing number → ask via AskUserQuestion: `Skip missing and proceed` / `Cancel`.
- Already `- [x]` → skip silently.
- Unparseable input → ask to clarify with the patterns above as options.

## Steps

1. **Read the three spec files** for context. Don't summarize them into the conversation.

2. **Resolve which task numbers to run** from argument + current `tasks.md` state.

3. **Delegate to `vibe-developer`** via the `Agent` tool with `subagent_type: "vibe-developer"`. Include:
   - Absolute path to `.vibepilot/spec/`
   - The exact task numbers (post-resolution)
   - The literal task lines for those numbers (so the agent doesn't re-parse `tasks.md`)
   - Reminders: (a) work in given order, deriving scope from `requirements.md` + `plan.md`; (b) flip each task's checkbox immediately on completion (durability); (c) stop on the first hard blocker rather than expanding scope.
   - Request return in this exact shape:
     - Completed: `<task numbers or "none">`
     - Files modified: `<paths or "none">`
     - Blocked: `<task number(s) and one-line reason, or "none">`
     - Notes: `<optional paragraph>`

4. **Relay the summary verbatim.** If blocked, surface prominently — don't bury. Then ask:
   - Blocked → how to proceed (resolve / skip / abort).
   - Clean → continue with next batch or hand off to `/vibepilot:review`.

   Don't auto-continue even if the user passed `all` — completing a batch is a natural checkpoint.

## Constraints

- Do not read, edit, or write source code yourself.
- Do not paste diffs or file contents — point at paths.
- Do not modify `requirements.md` or `plan.md`. Only the subagent's checkbox flips on `tasks.md` are allowed.
