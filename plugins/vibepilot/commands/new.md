---
description: Start a new spec workflow for a feature or task.
argument-hint: "[optional seed description]"
---

# /vibepilot:new

Initializes the four canonical files under `.vibepilot/work/`. Asks before clearing an in-progress dir. Use `/vibepilot:clean` instead if you want to wipe without re-initializing.

## Flow

1. `ls -la .vibepilot/work 2>/dev/null`.
   - Exists as a regular file (not a directory) → surface the unusual state and ask; don't blindly delete.
   - In-progress (any spec file has a `##` section beyond `# Title`, or `requirements.md` has a `> seed` line) → confirm via AskUserQuestion, naming the affected files: `Clear and start fresh (Recommended)` / `Cancel`. On cancel, output `.vibepilot/work/ left untouched. Run /vibepilot:clean if you want to wipe without re-initializing.` and stop. On clear, `rm -rf .vibepilot/work` and continue.

2. `mkdir -p .vibepilot/work` and write four placeholders:
   - `requirements.md` — `# Requirements`. If the user passed a free-form description, append a blank line then `> <description>` as the seed for `/vibepilot:clarify`.
   - `plan.md` — `# Plan`.
   - `tasks.md` — `# Tasks`.
   - `review.md` — `# Review`.

3. Report one line:
   - With seed: `.vibepilot/work/ is ready (seed captured). Next: /vibepilot:clarify`
   - Without seed: `.vibepilot/work/ is ready. Next: /vibepilot:clarify`

## Notes

- `# Title` headers stay English. The seed echo and the AskUserQuestion prompt match the user's conversation language.
- Only the four canonical files. No `.current`, no history dir, no extras.
- Don't overwrite in-progress content silently.
