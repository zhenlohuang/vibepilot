---
name: tasks
description: Decompose the current spec's `requirements.md` + `plan.md` into an ordered, checkbox-based task list at `.vibepilot/spec/tasks.md`. Trigger when the user runs `/vibepilot:tasks`, asks to break the plan into actionable steps, generate a checklist, regenerate the task list, or says they're ready to start executing — even if they only say something like "what's next" or "let's split this into steps".
---

# vibepilot:tasks — generate the task checklist

You are turning `requirements.md` + `plan.md` into a concrete, ordered task list at `.vibepilot/spec/tasks.md`. This is light work — read two files, write one — so do it directly in the main thread without invoking a subagent.

## Preconditions

- Both `.vibepilot/spec/requirements.md` and `.vibepilot/spec/plan.md` must exist with substantive content (real sections beyond the `# Title` header). If either is still a placeholder, tell the user which step to run first (`/vibepilot:clarify` or `/vibepilot:plan`) and stop.

## Language

Match the language used in `requirements.md` for the task text. The `# Tasks` header and the checkbox/number prefixes stay as-is so `/vibepilot:implement` can parse them deterministically.

## Steps

1. **Read both files** in full.

2. **Decide append vs. rewrite.** Treat `tasks.md` as already-substantive if it contains at least one `- [ ]` or `- [x]` task entry.

   If it does, ask via AskUserQuestion:
   - `Rewrite from scratch (Recommended if the plan changed materially)`
   - `Append new tasks at the end (Recommended for additive work)`

   When appending, **continue numbering from the existing maximum task number** — never reuse a number, even if some earlier items have been deleted. Reusing numbers breaks `/vibepilot:implement N` references and review scopes.

3. **Generate the task list.** Decompose the work into 3–15 ordered tasks. Each entry is a single imperative line:

   ```markdown
   - [ ] N. <Imperative one-liner describing the task>
   ```

   Rules:
   - Number tasks sequentially starting at 1 (or continuing from the existing max on append).
   - Order tasks so dependencies precede dependents — `vibe-developer` works through them top-to-bottom, so order *is* the dependency contract. No "Depends on" field is needed.
   - Each task should be small enough that a single `/vibepilot:implement N` invocation can finish it — roughly "one focused PR's worth".
   - Keep each line short and concrete (e.g., "Add CSV export endpoint at GET /reports/:id/export"). Don't restate the goal, files, or acceptance criteria — those belong in `requirements.md` and `plan.md`, which `vibe-developer` reads for context.
   - Don't invent tasks not grounded in `plan.md`. If the plan is unclear on something, write that as a task to revisit the plan rather than guessing at the work.

4. **Write the file.** Replace or append to `.vibepilot/spec/tasks.md`. Keep the `# Tasks` header at the top.

5. **Report.** Output the number of tasks written/added and one line:
   > Next: `/vibepilot:implement` to start executing tasks.

## Constraints

- Do **not** invoke the `Agent` tool — reading two files and writing one doesn't warrant delegation.
- Do **not** implement any code here. Tasks describe the work; `/vibepilot:implement` does it.
- Use GitHub-flavored markdown checkboxes (`- [ ]`). `/vibepilot:implement` will flip them to `- [x]` as it goes.
