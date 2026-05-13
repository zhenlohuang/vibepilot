---
name: implement
description: Execute one or more tasks from `.vibepilot/spec/tasks.md` by delegating to the `vibe-developer` subagent, which makes the actual code changes and ticks completed checkboxes. Trigger when the user runs `/vibepilot:implement` (optionally with a task number, range like `1-3`, comma list `1,3,5`, or `all`), asks to execute / build / code the spec, or says something like "let's start implementing" or "do task 2".
---

# vibepilot:implement — execute spec tasks

You are running one or more tasks from the current spec. Coding work — reading source files, editing them, running tests — is delegated to the `vibe-developer` subagent so the main thread doesn't fill up with file contents and diffs. Even small edits go through the subagent; the consistency matters more than the cost.

## Preconditions

- `.vibepilot/spec/{requirements,plan,tasks}.md` must all exist with substantive content. If any is missing or still a placeholder, tell the user which step to run first and stop.
- `.vibepilot/spec/tasks.md` must contain at least one `- [ ]` (unchecked) item. If everything is already `- [x]`, tell the user the implementation is complete and suggest `/vibepilot:review`.

## Argument parsing

The argument (everything after `/vibepilot:implement`) selects which tasks to run:

- Empty → the next single unchecked task (lowest-numbered `- [ ]`)
- A single number like `3` → just task 3
- A range like `1-3` → tasks 1 through 3 inclusive (must be ascending; a reversed range like `3-1` is invalid — ask the user to clarify)
- A comma-separated list like `1,3,5` → exactly those tasks
- `all` → every remaining unchecked task, in order

Validate the selection against the actual task numbers present in `tasks.md` before delegating:
- If a requested number doesn't exist, ask the user to confirm via AskUserQuestion (offer: `Skip missing and proceed` / `Cancel`).
- If a requested task is already `- [x]`, skip it silently — don't re-run completed work.
- If the input doesn't match any of the patterns above, ask the user to clarify with AskUserQuestion offering those patterns as options.

## Steps

1. **Read the three spec files** to gather context. Hold what you need to construct the delegation prompt — don't summarize them into the conversation.

2. **Resolve which task numbers to run** based on the argument and the current state of `tasks.md`.

3. **Delegate to `vibe-developer`** via the `Agent` tool with `subagent_type: "vibe-developer"`. The prompt should include:
   - The absolute path to `.vibepilot/spec/`
   - The exact task numbers to execute (post-resolution)
   - The literal task lines for those numbers (so the agent doesn't have to re-parse `tasks.md`)
   - A reminder to: (a) work the tasks in the given order, deriving scope from `requirements.md` + `plan.md`, (b) flip each completed task's `- [ ]` to `- [x]` in `tasks.md` immediately after finishing it — not in a batch at the end, since durability matters if the agent is interrupted, (c) stop and report on the first hard blocker rather than expanding scope to keep going
   - A request to return a summary in this exact shape:
     - Completed: `<list of task numbers, or "none">`
     - Files modified: `<list of paths, or "none">`
     - Blocked: `<task number(s) and one-line reason, or "none">`
     - Notes: `<one paragraph, optional>`

4. **Relay the summary** to the user verbatim. If any tasks are blocked, surface that prominently — don't bury it. Then ask:
   - If blocked: how to proceed (resolve the blocker / skip and move on / abort).
   - If clean: whether to continue with the next batch or hand off to `/vibepilot:review`.

   Don't auto-continue into another `/vibepilot:implement` round even if the user passed `all` — completing a batch is a natural checkpoint, and the user may want to inspect the changes before pressing on.

## Constraints

- Do **not** read, edit, or write any source code yourself. Even trivial-looking changes go through the subagent — this keeps responsibility (and the main thread) clean.
- Do **not** paste diffs or file contents into the conversation. Point at paths instead.
- Do **not** modify `requirements.md` or `plan.md`. The only spec edit allowed is the subagent flipping checkboxes in `tasks.md`.
