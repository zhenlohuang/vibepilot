---
name: clarify
description: Define the requirements of the current feature.
disable-model-invocation: true
---

# /vibepilot:clarify

Orchestrates the `vibe-clarifier` subagent (the same way `/vibepilot:plan` orchestrates `vibe-planner`) and is the only voice that talks to the user.

## Preconditions

- `.vibepilot/spec/` must exist. If not, tell the user to run `/vibepilot:new` first and stop.

## Steps

1. **Read `requirements.md`** once to capture text for the delegation prompt. Any `> seed` line written by `/vibepilot:new` is the starting hypothesis.

2. **Decide append vs. rewrite.** If the file has body beyond `# Requirements` and the seed line (real sections like `## Background` exist), ask via AskUserQuestion:
   - `Append / refine existing (Recommended)`
   - `Rewrite from scratch`

3. **Delegate to `vibe-clarifier` in draft mode** via the `Agent` tool with `subagent_type: "vibe-clarifier"`. Include in the parent prompt:
   - A line `mode: draft`
   - Absolute path to `.vibepilot/spec/`
   - Full text of `requirements.md`
   - The refine-or-rewrite choice
   - Instruction: return a proposed outline plus the user-only gaps, do NOT write any file in this mode.

4. **Ask the surfaced gaps.** If the draft returned gaps, run 1–N rounds of AskUserQuestion on them — up to 4 questions per round. Use the subagent's wording; do not invent generic questions. Skip this step entirely if the draft returned no gaps. If the user says "I don't know", propose a default or move it to Non-Goals — don't loop on the same question.

5. **Delegate to `vibe-clarifier` in finalize mode** via the `Agent` tool with `subagent_type: "vibe-clarifier"`. Include in the parent prompt:
   - A line `mode: finalize`
   - Absolute path to `.vibepilot/spec/`
   - The prior draft outline
   - Each gap with the user's answer
   - Instruction: write `.vibepilot/spec/requirements.md`, return a summary under 100 words — never paste the content.

6. **Relay the summary** and one line:
   > Next: `/vibepilot:plan` to design the implementation.

## Constraints

- Match the user's conversation language for body content. Section headers (`# Requirements`, `## Background`, etc.) stay in English so downstream skills can parse them.
- Do not read code, manifests, or `CLAUDE.md` / `README.md` yourself — that is the subagent's job.
- Do not draft requirements content yourself — surface it via `vibe-clarifier`'s draft mode.
- Do not paste the requirements into the conversation; only relay the summary.
- Questions to the user must come from the subagent's gap list and stay project-specific.
