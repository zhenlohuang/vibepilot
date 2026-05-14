---
description: Design how to implement the current feature and decompose it into a task checklist.
---

# /vibepilot:plan

Orchestrates `vibe-planner` in two passes. Produces both `plan.md` and `tasks.md` in one flow.

## Preconditions

- `.vibepilot/work/requirements.md` must have substantive content (sections beyond `# Requirements` and any `> seed`). Otherwise tell the user to run `/vibepilot:clarify` and stop.

## Flow

1. Read `requirements.md` and `tasks.md` once.

2. **Plan append/rewrite gate** — capture as `plan_mode`. If `plan.md` has any `##` section, ask via AskUserQuestion: `Refine existing plan (Recommended)` → `refine` / `Rewrite from scratch` → `rewrite`. Otherwise `plan_mode = rewrite`, skip the question.

3. **Tasks gate + max-number scan** — capture as `tasks_mode`. If `tasks.md` has any `- [ ]`/`- [x]` lines:
   - Compute `tasks_existing_max` by matching `^- \[[ x]\] (\d+)\.` and taking the highest number.
   - Ask AskUserQuestion. Default-recommend the option that mirrors `plan_mode` (rewrite→rewrite, refine→append): `Rewrite tasks from scratch` / `Append new tasks at the end`. Normalize the answer to `rewrite` or `append` (strip `(Recommended)` and prose — `vibe-planner` matches the bare token).
   - Otherwise `tasks_mode = rewrite`, `tasks_existing_max = 0`, skip.

4. **Draft pass** — delegate to `vibe-planner` (`subagent_type: "vibe-planner"`) with `mode: draft`, the absolute path to `.vibepilot/work/`, the `requirements.md` text, and `plan_mode`. Instruct it to return a candidate outline plus design uncertainties without writing any file. Tasks inputs aren't passed in draft mode.

5. If uncertainties came back, run 1–N rounds of AskUserQuestion (≤4 per round) using the subagent's wording verbatim.

6. **Finalize pass** — delegate to `vibe-planner` with `mode: finalize`, the draft outline, each uncertainty with the user's answer, `plan_mode`, `tasks_mode`, `tasks_existing_max`, and (when `tasks_mode: append`) the full body of existing `tasks.md`. Instruct it to write both `plan.md` and `tasks.md` and return one combined summary under 200 words.

7. Relay the summary. Surface: plan written (yes/no), task count and numbering range, primary architectural choice. If `plan.md` was not written, `tasks.md` is also skipped (planner enforces atomicity) — surface the planner's reason verbatim, recommend `/vibepilot:clarify`, and pause. Otherwise close with `Next: /vibepilot:implement to start executing tasks.`

## Notes

- Don't read code, run `git log`, or grep in the main thread — that's the subagent's job.
- Don't draft design content in the main thread; surface it via the draft pass.
- Don't write `tasks.md` yourself — only the planner does. The main thread only computes `tasks_existing_max` so the append-only numbering invariant is verifiable.
- Don't paste plan/task content into the conversation.
