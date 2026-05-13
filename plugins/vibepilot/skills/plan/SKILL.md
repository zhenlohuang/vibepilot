---
name: plan
description: Design how to implement the current feature and decompose it into a task checklist.
disable-model-invocation: true
---

# /vibepilot:plan

Orchestrates the `vibe-planner` subagent for codebase exploration, design drafting, and task decomposition. Produces both `plan.md` and `tasks.md` in one flow. The main thread is the only voice that talks to the user.

## Preconditions

- `.vibepilot/spec/requirements.md` must exist with substantive content (real sections beyond `# Requirements` and any `> seed` line). If still a placeholder, tell the user to run `/vibepilot:clarify` first and stop.

## Steps

1. **Read `requirements.md` and `tasks.md`** once. `requirements.md` text is captured for the delegation prompt; `tasks.md` content feeds the gate decision in step 3.

2. **Plan append/rewrite gate.** If `plan.md` has any `##` section (i.e. content beyond `# Plan`), ask via AskUserQuestion. Capture the answer as `plan_mode`.
   - `Refine existing plan (Recommended)`
   - `Rewrite from scratch`

3. **Tasks append/rewrite gate + max-number scan.** Using the `tasks.md` content read in step 1: if it contains any `- [ ]` or `- [x]` entry:
   - Compute `tasks_existing_max` by scanning lines that match `^- \[[ x]\] (\d+)\.` and taking the highest captured number.
   - Ask via AskUserQuestion. Capture the answer as `tasks_mode`. Default-recommend the option that matches `plan_mode`:
     - When `plan_mode == "Rewrite from scratch"`: `Rewrite tasks from scratch (Recommended)` / `Append new tasks at the end`.
     - When `plan_mode == "Refine existing plan"`: `Rewrite tasks from scratch` / `Append new tasks at the end (Recommended)`.
   - If `tasks.md` has no checkbox lines (placeholder or absent): set `tasks_mode = rewrite`, `tasks_existing_max = 0`, skip the question.
   - After the question (if asked), normalize the answer string into the token `rewrite` or `append` — strip the `(Recommended)` suffix and any prose. That token is what gets passed in step 6; passing the verbose label breaks `vibe-planner`'s exact match on `tasks_mode: rewrite|append`.

4. **Delegate to `vibe-planner` in draft mode** via the `Agent` tool with `subagent_type: "vibe-planner"`. Include in the parent prompt:
   - A line `mode: draft`
   - Absolute path to `.vibepilot/spec/`
   - Full text of `requirements.md`
   - The `plan_mode` choice
   - Instruction: return a candidate outline plus design uncertainties, do NOT write `plan.md` in this mode.

   Tasks inputs are NOT passed in draft mode — they only matter at finalize time.

5. **Ask the surfaced design questions.** If the draft returned uncertainties, run 1–N rounds of AskUserQuestion on them — up to 4 questions per round. Use the subagent's wording; do not invent generic questions. Skip this step entirely if the draft returned no uncertainties.

6. **Delegate to `vibe-planner` in finalize mode** via the `Agent` tool with `subagent_type: "vibe-planner"`. Include in the parent prompt:
   - A line `mode: finalize`
   - Absolute path to `.vibepilot/spec/`
   - Full text of `requirements.md`
   - The prior draft outline
   - Each uncertainty with the user's answer
   - `tasks_mode: rewrite` or `tasks_mode: append`
   - `tasks_existing_max: <int>`
   - When `tasks_mode: append`, the full body of existing `tasks.md` (it's small by design, so paste in full so the planner can avoid duplicating intent)
   - Instruction: write both `.vibepilot/spec/plan.md` and `.vibepilot/spec/tasks.md`, return one combined summary under 200 words — never paste the plan or task content into the response.

7. **Relay the combined summary** to the user. Surface: plan written (yes/no), tasks written (count and numbering range used), and the primary architectural choice. If `plan.md` was not written, `tasks.md` is also skipped (planner enforces atomicity) — surface the planner's reason verbatim, recommend `/vibepilot:clarify`, and pause. Otherwise close with: `Next: /vibepilot:implement to start executing tasks.`

## Constraints

- Do not read code, run `git log`, or `grep` yourself, and do not draft design choices yourself — surface them via `vibe-planner`'s draft mode.
- Do not write `tasks.md` yourself — only the planner does. Do compute `tasks_existing_max` from `tasks.md` before delegating finalize so the append-only numbering invariant is verifiable on the orchestrator side.
- Do not paste the plan or task content into the conversation; only relay the summary.
- Do not modify `requirements.md` or any other spec file.
