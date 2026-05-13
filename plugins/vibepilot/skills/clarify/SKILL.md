---
name: clarify
description: Interactively define and capture the requirements of the current spec into `.vibepilot/spec/requirements.md` — asks the user targeted questions (goal, user stories, acceptance criteria, constraints, non-goals). Trigger when the user runs `/vibepilot:clarify`, says they want to refine requirements / write a spec / clarify the task, or describes what they want to build in a way that calls for structured capture before designing — even if they only say something like "what should this feature actually do" or "let's nail down the requirements".
---

# vibepilot:clarify — define the requirements

You are clarifying the user's intent for the current spec and writing it to `.vibepilot/spec/requirements.md`. This work happens in the main thread on purpose — clarification needs back-and-forth with the user, and a subagent can't run a conversation. Don't try to delegate it.

## Preconditions

- `.vibepilot/spec/` must exist. If it doesn't, tell the user to run `/vibepilot:new` first and stop.

## Language

Match the language the user is using in this conversation. If the user is writing in Chinese (or any non-English language), write the file body in that language too. Section headers can stay in English (`# Requirements`, `## Background`, etc.) since they're structural markers — what should match the user's language is the body content they'll read and edit.

## Steps

1. **Load context.**
   - Read `.vibepilot/spec/requirements.md`. It may contain a seed `> ...` line written by `/vibepilot:new` — use that as the starting hypothesis when forming questions.
   - Read project root files that exist: `CLAUDE.md`, `README.md`, and the most relevant manifest (`package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` / etc.). The point is to make clarifying questions specific to *this project* rather than generic — citing actual modules, scripts, or conventions makes the conversation faster and the resulting spec better.

2. **Decide append vs. rewrite.** Treat the file as **already-substantive** if it has body content beyond the `# Requirements` header and any `> seed` line (i.e., real sections like `## Background` or `## User Stories` are already present).

   If already-substantive, ask via AskUserQuestion:
   - `Append / refine existing (Recommended)` — for follow-up clarification rounds
   - `Rewrite from scratch` — when the task has fundamentally changed

3. **Ask targeted questions** with AskUserQuestion. Run 1–4 rounds, each up to 4 questions. Cover (in order, only what's still unclear):
   - **Goal** — what problem is being solved, for whom?
   - **User stories** — who does what, in what context?
   - **Acceptance criteria** — concrete, testable conditions for "done"
   - **Constraints** — tech stack pins, performance/security/compat requirements
   - **Non-goals / out of scope** — what we are explicitly NOT doing in this task

   Questions should be **concrete and project-aware**: cite the project's actual patterns, files, or modules instead of asking in the abstract. A specific question gets a specific answer; a generic one wastes a round. Stop asking once you have enough to produce a useful spec — don't keep drilling for completeness once the shape is clear.

   If the user gives "I don't know" or skips a question, treat that as a signal: either propose a reasonable default and move on, or move the item to Non-Goals and surface that decision in the summary. Don't loop on the same question.

4. **Write `.vibepilot/spec/requirements.md`** with this section structure:

   ```markdown
   # Requirements

   ## Background
   <one short paragraph: problem and motivation>

   ## User Stories
   - As a <role>, I want <capability>, so that <benefit>.
   - ...

   ## Acceptance Criteria
   - [ ] <testable criterion>
   - ...

   ## Constraints
   - <constraint>
   - ...

   ## Non-Goals
   - <explicit out-of-scope item>
   - ...
   ```

   Drop a section entirely if there's genuinely nothing for it (e.g., no constraints surfaced) — better to omit than to pad with filler.

5. **Report.** Output a short summary (under 100 words) of what was captured, then one line:
   > Next: `/vibepilot:plan` to design the implementation.

## Constraints

- Do **not** invoke the `Agent` tool — clarification is an interactive activity that can't be delegated.
- Do **not** implement code or design architecture here. Those are the next two steps' jobs and getting ahead of them muddles the spec.
- Keep questions concrete and project-aware. Generic questions ("what's the goal?") get vague answers; project-specific ones ("should this reuse the auth middleware in `src/middleware/auth.ts` or introduce a new path?") get useful ones.
