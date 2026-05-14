---
description: Abandon the current spec and discard its progress.
---

# /vibepilot:clean

Removes `.vibepilot/work/` entirely. Use `/vibepilot:new` instead if you want to wipe **and** re-initialize.

## Flow

1. `ls -la .vibepilot/work 2>/dev/null`.
   - Doesn't exist → output `.vibepilot/work/ does not exist — nothing to clean.` and stop.
   - Exists as a regular file (not a directory) → surface the unusual state and ask; don't blindly delete.
   - Has in-progress content (any spec file has a `##` section beyond `# Title`, or `requirements.md` has a `> seed` line) → confirm via AskUserQuestion, naming the affected files: `Delete the spec (Recommended)` / `Cancel`. On cancel, output `Cancelled — .vibepilot/work/ left untouched.` and stop.

2. `rm -rf .vibepilot/work`.

3. Report one line: `.vibepilot/work/ removed. Run /vibepilot:new when you're ready to start a new spec.`

## Notes

- Only `.vibepilot/work/`. Never delete the parent `.vibepilot/`, source code, or git state.
- Don't recreate placeholder files — that's `/vibepilot:new`'s job.
