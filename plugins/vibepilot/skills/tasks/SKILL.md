---
name: tasks
description: Break the plan into actionable steps.
disable-model-invocation: true
---

# /vibepilot:tasks

Light work — read two files, write one. Runs in the main thread.

## Preconditions

- Both `.vibepilot/spec/requirements.md` and `.vibepilot/spec/plan.md` must have substantive content (real sections beyond `# Title`). If either is a placeholder, tell the user which step to run first and stop.

## Steps

1. **Read both files** in full.

2. **Decide append vs. rewrite.** If `tasks.md` already contains any `- [ ]` or `- [x]` entry, ask via AskUserQuestion:
   - `Rewrite from scratch (Recommended if the plan changed materially)`
   - `Append new tasks at the end (Recommended for additive work)`

   When appending, **continue numbering from the existing maximum** — never reuse a number, even if earlier items were deleted. Reusing breaks `/vibepilot:implement N` references and review scopes.

3. **Generate 3–15 ordered tasks**, one imperative line each:

   ```markdown
   - [ ] N. <Imperative one-liner>
   ```

   - Numbered sequentially from 1 (or continuing from the existing max).
   - Order = dependency contract — `vibe-developer` works top-to-bottom.
   - Each task ≈ one focused PR's worth.
   - Don't restate goal / files / acceptance criteria — those live in `requirements.md` and `plan.md`, which `vibe-developer` reads for context.
   - Don't invent tasks not grounded in `plan.md`. If the plan is unclear, write a task to revisit the plan rather than guessing.

4. **Write the file**, keeping `# Tasks` at the top.

5. **Report** task count written/added and one line:
   > Next: `/vibepilot:implement` to start executing tasks.

## Constraints

- Match `requirements.md`'s language for task text. Checkbox/number prefixes (`- [ ] N.`) stay canonical so `/vibepilot:implement` can parse them.
- Do not invoke the `Agent` tool.
- Do not write any code.
- Use `- [ ]` markdown checkboxes; `/vibepilot:implement` flips them to `- [x]`.
