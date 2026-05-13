---
name: clean
description: Abandon the current spec and discard its progress.
disable-model-invocation: true
---

# /vibepilot:clean

Removes `.vibepilot/spec/` entirely. Distinction from `/vibepilot:new`: `new` resets and starts over; `clean` abandons and leaves nothing.

## Preconditions

- None — the directory may or may not exist.

## Steps

1. **Check state.** Run `ls -la .vibepilot/spec 2>/dev/null`.
   - Doesn't exist → output `.vibepilot/spec/ does not exist — nothing to clean.` and stop.
   - Exists as a regular file (not a directory) → surface this unusual state and ask the user; do not blindly delete.
   - Has in-progress content (any of `requirements.md`, `plan.md`, `tasks.md`, `review.md` has body beyond its `# Title` header) → confirm via AskUserQuestion, naming the affected files:
     - `Delete the spec (Recommended)`
     - `Cancel`

     On cancel: output `Cancelled — .vibepilot/spec/ left untouched.` and stop.

2. **Delete.** `rm -rf .vibepilot/spec`.

3. **Report** one line: `.vibepilot/spec/ removed. Run /vibepilot:new when you're ready to start a new spec.`

## Constraints

- Do not invoke the `Agent` tool.
- Do not delete the parent `.vibepilot/` directory — only `.vibepilot/spec/`.
- Do not recreate placeholder files. Re-initializing is `/vibepilot:new`'s job.
- Do not touch source code, git state, or any other project file.
