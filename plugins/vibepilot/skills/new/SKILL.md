---
name: new
description: Initialize a fresh vibepilot spec workspace at `.vibepilot/spec/`, creating empty placeholder files (`requirements.md`, `plan.md`, `tasks.md`, `review.md`). Trigger when the user runs `/vibepilot:new`, says they want to start a new spec / new feature / new task, or describes a brand-new piece of work that should begin a spec — even if they only say something like "let's plan a new feature" or "kick off a new spec". Refuses to overwrite an in-progress spec; if one exists, point the user to `/vibepilot:clean` first.
---

# vibepilot:new — initialize a fresh spec

You are preparing a clean `.vibepilot/spec/` workspace so the user can proceed with `/vibepilot:clarify` next.

This skill is **non-destructive**: if an in-progress spec already exists, it refuses and points the user at `/vibepilot:clean`. Splitting the responsibilities means the user must *explicitly* throw away previous work — `new` will never silently destroy a draft, even with confirmation.

## Inputs

- Optional free-form task description from the user (everything after `/vibepilot:new`).
- The current working directory's `.vibepilot/spec/` directory (may or may not exist).

## Steps

1. **Check existing state.** Run `ls -la .vibepilot/spec 2>/dev/null` (or equivalent).

   Treat the workspace as **in-progress** if any of `requirements.md`, `plan.md`, `tasks.md`, `review.md` has body content beyond its single `# Title` header line. A seed `> ...` line counts too — it represents user intent we don't want to clobber.

   If in-progress work is detected, stop without modifying anything. Output one short message naming the affected files so the user knows what's there:
   > `.vibepilot/spec/ already contains in-progress work (<list the affected files>). Run /vibepilot:clean to discard it, then /vibepilot:new again to start fresh.`

   If `.vibepilot/spec` exists as a regular file rather than a directory, that's an unusual state — surface it to the user and ask how to proceed rather than touching it.

2. **Initialize the directory.** Once the safety check has passed (directory missing, empty, or contains only placeholders):
   ```bash
   mkdir -p .vibepilot/spec
   ```

3. **Write four placeholder files** at `.vibepilot/spec/`:
   - `requirements.md` — single line `# Requirements`. If the user passed a task description on the command line, add a blank line and then `> <description>` underneath as a seed for `/vibepilot:clarify` to pick up.
   - `plan.md` — single line `# Plan`.
   - `tasks.md` — single line `# Tasks`.
   - `review.md` — single line `# Review`.

   Write all four unconditionally. The in-progress check in step 1 already ruled out anything worth preserving, and overwriting placeholder-with-placeholder is harmless.

4. **Report.** Output one short line, then stop. Mention if a seed was captured so the user knows it'll feed into clarify:
   - With seed: `.vibepilot/spec/ is ready (seed description captured). Next: /vibepilot:clarify`
   - Without seed: `.vibepilot/spec/ is ready. Next: /vibepilot:clarify`

## Constraints

- Do **not** invoke the `Agent` tool. This work is trivial and belongs in the main thread.
- Do **not** delete or overwrite in-progress content. Deletion is `/vibepilot:clean`'s job — keep the responsibilities split so users explicitly opt in to destruction.
- Do **not** read or analyze project files beyond what's needed to check `.vibepilot/spec/` state.
- Do **not** create any other files (no `id`, no `.current`, no history dir). Keep the workspace minimal — only the four canonical spec files.
