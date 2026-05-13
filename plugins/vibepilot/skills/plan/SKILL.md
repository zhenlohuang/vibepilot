---
name: plan
description: Design how to implement the current feature.
disable-model-invocation: true
---

# /vibepilot:plan

Orchestrates the `vibe-planner` subagent for codebase exploration and design drafting, and is the only voice that talks to the user.

## Preconditions

- `.vibepilot/spec/requirements.md` must exist with substantive content (real sections beyond `# Requirements` and any `> seed` line). If still a placeholder, tell the user to run `/vibepilot:clarify` first and stop.

## Steps

1. **Read `requirements.md`** once to verify substance and capture text for the delegation prompt.

2. **Decide append vs. rewrite.** If `plan.md` has body beyond `# Plan` (any real section like `## Architecture Overview`), ask via AskUserQuestion:
   - `Refine existing plan (Recommended)`
   - `Rewrite from scratch`

3. **Delegate to `vibe-planner` in draft mode** via the `Agent` tool with `subagent_type: "vibe-planner"`. Include in the parent prompt:
   - A line `mode: draft`
   - Absolute path to `.vibepilot/spec/`
   - Full text of `requirements.md`
   - The refine-or-rewrite choice
   - Instruction: return a candidate outline plus design uncertainties, do NOT write `plan.md` in this mode.

4. **Ask the surfaced design questions.** If the draft returned uncertainties, run 1–N rounds of AskUserQuestion on them — up to 4 questions per round. Use the subagent's wording; do not invent generic questions. Skip this step entirely if the draft returned no uncertainties.

5. **Delegate to `vibe-planner` in finalize mode** via the `Agent` tool with `subagent_type: "vibe-planner"`. Include in the parent prompt:
   - A line `mode: finalize`
   - Absolute path to `.vibepilot/spec/`
   - Full text of `requirements.md`
   - The prior draft outline
   - Each uncertainty with the user's answer
   - Instruction: write `.vibepilot/spec/plan.md`, return a summary under 200 words — never paste the plan into the response.

6. **Relay the summary** to the user. If the agent reports it couldn't write the file (unresolved question, missing context), surface its reason verbatim and pause — usually the right next move is `/vibepilot:clarify`. Then ask whether to iterate or proceed to `/vibepilot:tasks`.

## Constraints

- Do not read code, run `git log`, or `grep` yourself, and do not draft design choices yourself — surface them via `vibe-planner`'s draft mode.
- Do not paste the plan into the conversation; only relay the summary.
- Do not modify `requirements.md` or any other spec file.
