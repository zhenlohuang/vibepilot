---
name: implement
description: Build the code for the remaining tasks of the current spec.
disable-model-invocation: true
argument-hint: "[optional scope]"
---

# /vibepilot:implement

All coding work is delegated to `vibe-developer` — even small edits, for consistency. The skill is argumentless by default, but adapts to whatever scope the user describes.

## Preconditions

- `.vibepilot/work/{requirements,plan,tasks}.md` must all have substantive content. If any is missing or a placeholder, tell the user which step to run first and stop.
- `tasks.md` must contain at least one `- [ ]`. If everything is `- [x]`, tell the user the implementation is complete and suggest `/vibepilot:review`.

## Default scope

No argument → run every `- [ ]` task in `tasks.md`, in file order. Stop on the first hard blocker.

## Adapting to user input

If the user passes anything — a task number, a range, a list, a free-form description like "just the auth tasks" or "redo task 2" — interpret it against `tasks.md` and resolve the task set yourself. Don't force the user into a specific arg syntax; trust their framing.

Once resolved, validate the set against actual task numbers in `tasks.md`:
- Missing / nonexistent number → ask via AskUserQuestion: `Skip missing and proceed` / `Cancel`.
- Already `- [x]` → skip silently unless the user explicitly asked to redo it.
- Resolution genuinely ambiguous → confirm via AskUserQuestion; otherwise proceed with the obvious reading.

## Steps

1. **Read the three spec files** for context. Don't summarize them into the conversation.

2. **Resolve the task set.** No arg/hint → every `- [ ]` line in `tasks.md`, in file order. Otherwise → apply Adapting to user input. Capture both the task numbers and the literal task lines.

3. **Delegate to `vibe-developer`** via the `Agent` tool with `subagent_type: "vibe-developer"`. Include:
   - Absolute path to `.vibepilot/work/`
   - The resolved task numbers, in order
   - The literal task lines for those numbers (so the agent doesn't re-parse `tasks.md`)
   - Reminders: (a) work in given order, deriving scope from `requirements.md` + `plan.md`; (b) flip each task's checkbox immediately on completion (durability); (c) stop on the first hard blocker rather than expanding scope.
   - Request return in this exact shape:
     - Completed: `<task numbers or "none">`
     - Files modified: `<paths or "none">`
     - Blocked: `<task number(s) and one-line reason, or "none">`
     - Notes: `<optional paragraph>`

4. **Relay the summary verbatim.** If blocked, surface prominently — don't bury. Then ask:
   - Blocked → how to proceed (resolve and re-run `/vibepilot:implement` / abort).
   - Clean with remaining `- [ ]` → ask whether to run `/vibepilot:implement` again or pause.
   - All tasks `- [x]` → suggest `/vibepilot:review`.

   Don't auto-continue after the subagent returns — completing a batch is a natural checkpoint.

## Constraints

- Do not read, edit, or write source code yourself.
- Do not paste diffs or file contents — point at paths.
- Do not modify `requirements.md` or `plan.md`. Only the subagent's checkbox flips on `tasks.md` are allowed.
