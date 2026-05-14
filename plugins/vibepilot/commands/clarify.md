---
description: Define the requirements of the current feature.
---

# /vibepilot:clarify

Orchestrates `vibe-clarifier` in two passes. The main thread is the only voice that talks to the user.

## Preconditions

- `.vibepilot/work/` must exist. Otherwise tell the user to run `/vibepilot:new` and stop.

## Flow

1. Read `requirements.md`. Any `> seed` line is a starting hypothesis, not authoritative.

2. If `requirements.md` already has any `##` section, ask via AskUserQuestion: `Append / refine existing (Recommended)` / `Rewrite from scratch`.

3. **Draft pass** — delegate to `vibe-clarifier` (`subagent_type: "vibe-clarifier"`) with `mode: draft`, the absolute path to `.vibepilot/work/`, the current `requirements.md` text, and the refine/rewrite choice. Instruct it to return a proposed outline and user-only gaps without writing any file.

4. If gaps came back, run 1–N rounds of AskUserQuestion (≤4 per round) using the subagent's wording verbatim. If the user says "I don't know", propose a default or move it to Non-Goals — don't loop.

5. **Finalize pass** — delegate to `vibe-clarifier` with `mode: finalize`, the draft outline, and each gap with the user's answer. Instruct it to write `requirements.md` and return a summary under 100 words.

6. Relay the summary and append: `Next: /vibepilot:plan to design the implementation.`

## Notes

- Don't read code/manifests yourself — that's the subagent's job.
- Don't draft requirements content or invent generic gap questions in the main thread.
- Don't paste `requirements.md` content back into the conversation.
- Section headers (`# Requirements`, `## Background`, …) stay English so downstream commands can parse them; body matches the user's conversation language.
