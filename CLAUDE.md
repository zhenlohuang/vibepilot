# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

vibepilot is a **Claude Code plugin**, not a runtime application. Everything that "runs" is markdown frontmatter + body that the Claude Code harness loads. There is no build step, no test suite, no linter, no package manager — edits are live the moment Claude Code reloads the plugin.

Distribution model is a **self-hosted single-plugin marketplace**: the root `.claude-plugin/marketplace.json` advertises one plugin at `./plugins/vibepilot/`.

## Local development loop

Inside Claude Code, after changing any file under `plugins/vibepilot/`:

```text
/plugin reload
```

To install the plugin from a working copy for testing:

```text
/plugin marketplace add /absolute/path/to/vibepilot
/plugin install vibepilot@vibepilot
```

All user-facing commands are namespaced as `/vibepilot:<skill-name>` (the skill directory name becomes the command name). There are no shell tests to run — verification means invoking the commands inside Claude Code.

## Architecture

```
.claude-plugin/marketplace.json   # marketplace metadata; points at plugins/vibepilot
plugins/vibepilot/
├── .claude-plugin/plugin.json    # plugin manifest (name, version, author)
├── agents/*.md                   # vibe-clarifier, vibe-planner, vibe-developer, vibe-reviewer
├── commands/                     # unused; all user entries live under skills/
└── skills/<name>/SKILL.md        # one directory per slash command
```

Each `SKILL.md` carries `disable-model-invocation: true` in its frontmatter, so the slash commands are **manual-only** — Claude won't auto-trigger them on intent. The `description` field is therefore only used in the command-list UI; keep it to a single line about what the command does, not when to trigger it.

### The spec-driven workflow

The six skills implement a five-step pipeline whose only durable state is `.vibepilot/spec/` in the user's project (NOT in this repo):

```
new → clarify → plan → implement → review     (+ clean to wipe)
```

Four canonical files live there: `requirements.md`, `plan.md`, `tasks.md`, `review.md`. They form an authority hierarchy:

- `requirements.md` = **what** (acceptance criteria are the ground truth for "done")
- `plan.md` = **how** (file targets, approach, reuse — governs scope of each task)
- `tasks.md` = **imperative pointers** (intentionally terse one-liners; not self-sufficient)

Task lines deliberately do not duplicate context — `vibe-developer` reads `requirements.md` + `plan.md` to know what each task actually means.

### Main-thread vs. subagent split

This is the single most important design rule and it must be preserved. The main thread is always the orchestrator and the only voice that talks to the user; subagents do the reads, drafts, and writes in isolated context.

| Skill | Main-thread orchestration | Subagent work |
|---|---|---|
| `new`, `clean` | Trivial file ops with no heavy reads/drafts — runs entirely on main thread | — |
| `clarify` | Append/rewrite gate, asks the user the surfaced gaps via `AskUserQuestion` | `vibe-clarifier` reads project context, drafts a requirements outline, then writes `requirements.md` |
| `plan` | Plan append/rewrite gate, tasks append/rewrite gate (computes existing max number to preserve append-only IDs), asks the user the surfaced design questions via `AskUserQuestion` | `vibe-planner` explores the codebase, drafts a plan outline, then writes both `plan.md` and `tasks.md` |
| `implement` | Picks task numbers, relays summary | `vibe-developer` reads spec + source, edits code, flips checkboxes in `tasks.md` |
| `review` | Scope selection, relays summary | `vibe-reviewer` reads diffs + spec, writes `review.md` |

Subagents (`vibe-clarifier`, `vibe-planner`, `vibe-developer`, `vibe-reviewer`) must:
- Write their output to a file under `.vibepilot/spec/` and return **only a short summary** (< 200 words) to the parent.
- Never paste plan/diff/file contents into the response.
- Have strictly bounded write scope (clarifier writes only `requirements.md`; planner writes only `plan.md` and `tasks.md`; developer writes source code + flips checkboxes in `tasks.md`; reviewer writes only `review.md`).

`clarify` and `plan` use a two-phase `mode: draft` / `mode: finalize` pattern. The mode is passed as a plain-text line in the parent prompt. In `mode: draft` the subagent reads context, returns an outline plus the gaps or design uncertainties only the user can resolve, and writes nothing. The main thread then runs 1–N rounds of `AskUserQuestion` on those gaps (skipped entirely if the draft returned none). The main thread re-delegates in `mode: finalize` with the prior outline plus the user's answers, and only then does the subagent write the spec file. This matches Claude Code's own `/plan` (plan mode) orchestration shape.

When editing or adding skills, follow this split. Even small code edits go through `vibe-developer` — consistency matters more than the cost.

### Invariants worth preserving

- **One spec at a time.** `/vibepilot:new` refuses to overwrite in-progress work; the user must `/vibepilot:clean` first. Keep `new` non-destructive and `clean` non-recreating — the split is deliberate.
- **Task numbers are append-only.** When `plan` appends to an existing `tasks.md`, it continues from the existing max — never reuses numbers. The main thread computes `tasks_existing_max` and passes it into `vibe-planner`'s finalize delegation. `/vibepilot:implement N` and review scopes rely on this.
- **Checkbox flips are durable.** `vibe-developer` flips `- [ ]` → `- [x]` in `tasks.md` immediately after each task completes, not in a batch — interruption resilience.
- **Language match.** Skill bodies match the user's conversation language (and the language of `requirements.md` for downstream files). Section headers (`# Plan`, `## Background`) and checkbox prefixes stay in English / canonical form so downstream skills can parse them deterministically.
- **No drive-by refactors.** Developer subagent stays inside the scope each task + plan describes; if scope blows out, it stops and marks the task blocked rather than expanding.

## Editing skill prompts

When adjusting a `SKILL.md`:
- Keep `disable-model-invocation: true` in the frontmatter. vibepilot's commands are manual-only; don't add auto-trigger keyword lists to `description` — one-line "what it does" is enough.
- Runtime rules (Language matching, hard constraints) must live in the `SKILL.md` itself — **not** in this `CLAUDE.md`. End users who install the plugin don't load this file; only contents of `SKILL.md` and `agents/*.md` reach them.
- Preconditions sections matter: each skill should refuse and point to the prerequisite skill when inputs aren't ready, rather than silently doing partial work.
- Keep the constraints list — it's how the skill resists being asked to overreach (e.g., `plan` not reading code itself; `implement` not editing `plan.md`).
