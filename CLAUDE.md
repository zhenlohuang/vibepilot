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
├── agents/*.md                   # vibe-planner, vibe-developer, vibe-reviewer
├── commands/                     # currently empty; skills double as commands
└── skills/<name>/SKILL.md        # one directory per slash command
```

Each `SKILL.md` and each agent `.md` has YAML frontmatter (`name`, `description`, sometimes `tools` / `model`) followed by the prompt body. The `description` field is what Claude Code matches against user intent to auto-trigger the skill — keep it explicit about *when* to trigger, not just *what* it does.

### The spec-driven workflow

The seven skills implement a six-step pipeline whose only durable state is `.vibepilot/spec/` in the user's project (NOT in this repo):

```
new → clarify → plan → tasks → implement → review     (+ clean to wipe)
```

Four canonical files live there: `requirements.md`, `plan.md`, `tasks.md`, `review.md`. They form an authority hierarchy:

- `requirements.md` = **what** (acceptance criteria are the ground truth for "done")
- `plan.md` = **how** (file targets, approach, reuse — governs scope of each task)
- `tasks.md` = **imperative pointers** (intentionally terse one-liners; not self-sufficient)

Task lines deliberately do not duplicate context — `vibe-developer` reads `requirements.md` + `plan.md` to know what each task actually means.

### Main-thread vs. subagent split

This is the single most important design rule and it must be preserved:

| Skill | Where the work runs | Why |
|---|---|---|
| `new`, `clarify`, `tasks`, `clean` | **Main thread** | Trivial file ops or require back-and-forth with the user |
| `plan`, `implement`, `review` | **Delegated to subagent** | Reads source files / diffs — would pollute main context |

Subagents (`vibe-planner`, `vibe-developer`, `vibe-reviewer`) must:
- Write their output to a file under `.vibepilot/spec/` and return **only a short summary** (< 200 words) to the parent.
- Never paste plan/diff/file contents into the response.
- Have strictly bounded write scope (planner writes only `plan.md`; developer writes source code + flips checkboxes in `tasks.md`; reviewer writes only `review.md`).

When editing or adding skills, follow this split. Even small code edits go through `vibe-developer` — consistency matters more than the cost.

### Invariants worth preserving

- **One spec at a time.** `/vibepilot:new` refuses to overwrite in-progress work; the user must `/vibepilot:clean` first. Keep `new` non-destructive and `clean` non-recreating — the split is deliberate.
- **Task numbers are append-only.** When `tasks` appends to an existing list, it continues from the existing max — never reuses numbers. `/vibepilot:implement N` and review scopes rely on this.
- **Checkbox flips are durable.** `vibe-developer` flips `- [ ]` → `- [x]` in `tasks.md` immediately after each task completes, not in a batch — interruption resilience.
- **Language match.** Skill bodies match the user's conversation language (and the language of `requirements.md` for downstream files). Section headers (`# Plan`, `## Background`) and checkbox prefixes stay in English / canonical form so downstream skills can parse them deterministically.
- **No drive-by refactors.** Developer subagent stays inside the scope each task + plan describes; if scope blows out, it stops and marks the task blocked rather than expanding.

## Editing skill prompts

When adjusting a `SKILL.md`:
- The `description` frontmatter is auto-trigger material — be specific about when the skill should fire (and when it shouldn't), since Claude Code uses it to disambiguate.
- Preconditions sections matter: each skill should refuse and point to the prerequisite skill when its inputs aren't ready, rather than silently doing partial work.
- Keep the constraints list — it's how the skill resists being asked to overreach (e.g., `plan` not reading code itself; `implement` not editing `plan.md`).
