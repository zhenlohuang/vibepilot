---
name: plan
description: Design the implementation approach for the current spec and write it to `.vibepilot/spec/plan.md`. Delegates codebase exploration and architectural design to the `vibe-planner` subagent so the main conversation stays uncluttered. Trigger when the user runs `/vibepilot:plan`, says they want to design / architect / plan the implementation, asks "how should we build this" against the captured requirements, or wants to re-plan after the spec has changed — even if they don't explicitly say the word "plan".
---

# vibepilot:plan — design the implementation

You are producing an implementation plan for the current spec. The heavy lifting — codebase exploration, architectural reasoning, enumerating files — is delegated to the `vibe-planner` subagent. Doing it inline would pollute the main thread with file contents the user doesn't need to see, and would crowd out room for the review conversation later.

## Preconditions

- `.vibepilot/spec/requirements.md` must exist with substantive content. **Substantive** means body content beyond the `# Requirements` header and any `> seed` line — at least one real section (e.g., `## Background`, `## User Stories`). If the file is still a placeholder, tell the user to run `/vibepilot:clarify` first and stop.

## Steps

1. **Read requirements.** Read `.vibepilot/spec/requirements.md` once to verify it's substantive and to capture its content for the delegation prompt.

2. **Decide append vs. rewrite.** Treat `plan.md` as already-substantive if it has body content beyond the `# Plan` header (any real section like `## Architecture Overview` is present). If so, ask via AskUserQuestion:
   - `Refine existing plan (Recommended)`
   - `Rewrite from scratch`

3. **Delegate to `vibe-planner`** via the `Agent` tool with `subagent_type: "vibe-planner"`. The prompt should include:
   - The absolute path to the spec directory (`.vibepilot/spec/`)
   - The full text of `requirements.md`
   - Whether to refine the existing `plan.md` or rewrite it
   - An explicit instruction: the deliverable is the file `.vibepilot/spec/plan.md` — NOT a plan in the agent's response. The agent writes the file and returns a short summary (under 200 words) of the key decisions.

4. **Relay the agent's summary** to the user, then ask whether they want to iterate (another `/vibepilot:plan` round) or proceed to `/vibepilot:tasks`.

   If the agent reports it could not write the file (e.g., blocked by an unresolved question, missing context, or an explicit failure), surface its reason verbatim and pause — don't retry blindly. The right next move is usually a brief follow-up `/vibepilot:clarify` to fill the gap.

## Constraints

- Do **not** read code files, run `git log`, or `grep` yourself — the subagent handles exploration. Pulling that content into the main thread defeats the purpose of delegating.
- Do **not** paste the plan content into the conversation. Only relay the agent's summary.
- Do **not** modify `requirements.md` or any other spec file. This skill only triggers the planner; only the planner writes `plan.md`.
