---
description: Build the code for the remaining tasks of the current spec.
argument-hint: "[optional scope]"
---

# /vibepilot:implement

All coding is delegated to `vibe-developer`. Argumentless by default; adapts to whatever scope the user describes.

## Preconditions

- `.vibepilot/work/{requirements,plan,tasks}.md` must all be substantive. Otherwise tell the user which step to run first and stop.
- `tasks.md` must contain at least one `- [ ]`. If everything is `- [x]`, suggest `/vibepilot:review` and stop.

## Flow

1. Read the three spec files for context; don't summarize them into the conversation.

2. **Resolve the task set.**
   - No arg → every `- [ ]` line in `tasks.md`, in file order.
   - With arg (task number, range, list, free-form like "just the auth tasks", "redo task 2") → interpret against `tasks.md`. Validate against actual task numbers:
     - Missing/nonexistent number → AskUserQuestion: `Skip missing and proceed` / `Cancel`.
     - Already `- [x]` → skip silently unless the user explicitly said "redo".
     - Genuinely ambiguous → confirm; otherwise proceed with the obvious reading.

3. **Delegate to `vibe-developer`** (`subagent_type: "vibe-developer"`) with:
   - Absolute path to `.vibepilot/work/`
   - The resolved task numbers in order, plus the literal task lines (so the agent doesn't re-parse `tasks.md`)
   - Reminders: (a) derive scope from `requirements.md` + `plan.md`; (b) flip each checkbox immediately on completion; (c) stop on the first hard blocker rather than expanding scope.
   - Required return shape:
     - Completed: `<task numbers or "none">`
     - Files modified: `<paths or "none">`
     - Blocked: `<task number(s) and one-line reason, or "none">`
     - Notes: `<optional paragraph>`

4. Relay the summary verbatim. If blocked, surface prominently. Then ask:
   - Blocked → how to proceed (resolve and re-run / abort).
   - Remaining `- [ ]` → re-run `/vibepilot:implement` or pause.
   - All `- [x]` → suggest `/vibepilot:review`.

   Don't auto-continue — a finished batch is a checkpoint.

## Notes

- Don't read, edit, or write source code in the main thread.
- Don't modify `requirements.md` or `plan.md`. Only the subagent's checkbox flips on `tasks.md` are allowed.
- Don't paste diffs or file contents — point at paths.
