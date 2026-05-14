---
name: new
description: Start a new spec workflow for a feature or task.
disable-model-invocation: true
---

# /vibepilot:new

On a non-empty work dir, asks before clearing — then initializes the four canonical files. Use `/vibepilot:clean` instead if you want to wipe without re-initializing.

## Preconditions

- None — works in any directory.

## Steps

1. **Check existing state.** Run `ls -la .vibepilot/work 2>/dev/null`. Treat as in-progress if any of `requirements.md`, `plan.md`, `tasks.md`, `review.md` has any `##` section beyond its `# Title` (or `requirements.md` has a `> seed` line).

   If `.vibepilot/work` exists as a regular file (not a directory), surface this unusual state to the user and ask how to proceed — do not blindly delete.

   If in-progress, confirm via AskUserQuestion, naming the affected files:
   - `Clear and start fresh (Recommended)`
   - `Cancel`

   On `Cancel`: output `.vibepilot/work/ left untouched. Run /vibepilot:clean if you want to wipe without re-initializing.` and stop.

   On `Clear and start fresh`: `rm -rf .vibepilot/work` and continue.

2. **Initialize.** `mkdir -p .vibepilot/work`.

3. **Write four placeholders** at `.vibepilot/work/`:
   - `requirements.md` — `# Requirements`. If the user passed a free-form description after `/vibepilot:new`, append a blank line and then `> <description>` as a seed for `/vibepilot:clarify`.
   - `plan.md` — `# Plan`.
   - `tasks.md` — `# Tasks`.
   - `review.md` — `# Review`.

4. **Report** one line:
   - With seed: `.vibepilot/work/ is ready (seed captured). Next: /vibepilot:clarify`
   - Without seed: `.vibepilot/work/ is ready. Next: /vibepilot:clarify`

## Constraints

- Match the user's conversation language when echoing the seed and the AskUserQuestion prompt. The `# Title` headers stay in English so downstream skills can parse them.
- Do not invoke the `Agent` tool.
- Do not overwrite in-progress content silently — wipe only after explicit `Clear and start fresh` confirmation. Use `/vibepilot:clean` if the user wants to wipe without re-initializing.
- Do not create extra files (no `.current`, no history dir). Only the four canonical files.
