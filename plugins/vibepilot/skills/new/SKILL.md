---
name: new
description: Start a new spec workflow for a feature or task.
disable-model-invocation: true
---

# /vibepilot:new

Non-destructive: refuses to overwrite in-progress work. Use `/vibepilot:clean` to discard first.

## Preconditions

- None — works in any directory.

## Steps

1. **Check existing state.** Run `ls -la .vibepilot/spec 2>/dev/null`. Treat as in-progress if any of `requirements.md`, `plan.md`, `tasks.md`, `review.md` has any `##` section beyond its `# Title` (or `requirements.md` has a `> seed` line). If in-progress, stop and tell the user to run `/vibepilot:clean` first, naming the affected files.

   If `.vibepilot/spec` exists as a regular file (not a directory), surface this unusual state to the user and ask how to proceed.

2. **Initialize.** `mkdir -p .vibepilot/spec`.

3. **Write four placeholders** at `.vibepilot/spec/`:
   - `requirements.md` — `# Requirements`. If the user passed a free-form description after `/vibepilot:new`, append a blank line and then `> <description>` as a seed for `/vibepilot:clarify`.
   - `plan.md` — `# Plan`.
   - `tasks.md` — `# Tasks`.
   - `review.md` — `# Review`.

4. **Report** one line:
   - With seed: `.vibepilot/spec/ is ready (seed captured). Next: /vibepilot:clarify`
   - Without seed: `.vibepilot/spec/ is ready. Next: /vibepilot:clarify`

## Constraints

- Match the user's conversation language when echoing the seed. The `# Title` headers stay in English so downstream skills can parse them.
- Do not invoke the `Agent` tool.
- Do not overwrite in-progress content — that's `/vibepilot:clean`'s job.
- Do not create extra files (no `.current`, no history dir). Only the four canonical files.
