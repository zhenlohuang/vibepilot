---
name: clean
description: Remove the vibepilot spec workspace at `.vibepilot/spec/` entirely, leaving no placeholder files behind. Trigger when the user runs `/vibepilot:clean`, asks to delete / wipe / remove / clean up the spec, wants to abandon the current spec without starting a new one, or says something like "throw away the spec", "remove .vibepilot/spec", or "reset everything". If the user actually wants a fresh spec to work on next, prefer `/vibepilot:new` — `clean` leaves nothing behind on purpose.
---

# vibepilot:clean — remove the spec workspace

You are removing `.vibepilot/spec/` entirely. Unlike `/vibepilot:new`, this skill does NOT recreate placeholder files — after it runs, there is no spec to work with until the user runs `/vibepilot:new` again. The distinction matters: `new` is "reset and start over"; `clean` is "abandon and leave nothing".

The job is intentionally trivial: protect the user from accidentally destroying in-progress work, then delete the directory. Don't read project source code, don't invoke subagents.

## Inputs

- The current working directory's `.vibepilot/spec/` directory (may or may not exist).

## Steps

1. **Check existing state.** Run `ls -la .vibepilot/spec 2>/dev/null`.

   If `.vibepilot/spec/` doesn't exist, output one line and stop: `.vibepilot/spec/ does not exist — nothing to clean.`

   Treat the workspace as **in-progress** if any of `requirements.md`, `plan.md`, `tasks.md`, `review.md` has body content beyond its single `# Title` header line (i.e., real content, not just placeholders). A quick `wc -l` or short read is enough to tell.

   If in-progress work is detected, confirm with AskUserQuestion before deleting. Name the affected files so the user can judge what they'd lose:
   - `Delete the spec (Recommended)`
   - `Cancel`

   If the user cancels, stop with one line: `Cancelled — .vibepilot/spec/ left untouched.`

   If `.vibepilot/spec` exists as a regular file rather than a directory, that's an unusual state — surface it to the user and ask how to proceed instead of blindly deleting.

2. **Delete the directory.** Once confirmed (or if the directory was empty / all-placeholders):
   ```bash
   rm -rf .vibepilot/spec
   ```

3. **Report.** Output one short line, then stop:
   `.vibepilot/spec/ removed. Run /vibepilot:new when you're ready to start a new spec.`

## Constraints

- Do **not** invoke the `Agent` tool. This work is trivial and belongs in the main thread.
- Do **not** delete the parent `.vibepilot/` directory itself — it may hold other vibepilot data in the future. Only remove `.vibepilot/spec/`.
- Do **not** recreate placeholder files. Re-initializing is `/vibepilot:new`'s job — keeping the distinction clear matters so users can choose intent.
- Do **not** touch source code, git state, or any other project file.
