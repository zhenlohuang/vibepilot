---
name: spec-driven-development
description: Spec-driven development — capture WHY before HOW, decompose HOW into a checklist, then execute and audit against both. Use this skill whenever `.vibepilot/work/` is present in the repo, when reading or editing `requirements.md` / `plan.md` / `tasks.md` / `review.md` files, when the user mentions specs, requirements, acceptance criteria, or task checklists, or when the user is doing requirements → plan → implement → review style work in any project — even when they don't name the workflow.
---

# Spec-driven development

A small, opinionated workflow for "think before you build" — capture **why** before **how**, decompose **how** into a checklist, then execute and audit against both. The vibepilot plugin ships a concrete implementation under `.vibepilot/work/`; the principles apply more broadly.

## 1. Principles

- **One spec at a time.** Work is single-threaded. Interleaving multiple specs in the same workspace breaks every invariant below.
- **The spec is the source of truth.** Requirements, design, and progress live in spec files — not in commit messages, PR descriptions, or chat scrollback. If a fact only exists in conversation, it isn't durable.
- **Sequential flow with explicit gates.** `requirements → plan → tasks → implement → review`. Each downstream artifact requires substantive content in its predecessor — placeholders block progression.
- **Checkboxes are an invariant.** Tasks use `- [ ]` (pending) and `- [x]` (done). The set of `- [x]` lines is **monotonic** — once flipped, don't flip back; if a task needs redo, add a new one. Task numbering is **append-only across rewrites**: max task number never decreases.
- **Idempotent, resumable, stateful.** Any command can be re-run. State persists across sessions and SIGINT; the next invocation reads the files and picks up. Never store progress in volatile memory.
- **No silent destructive writes.** Substantive content must not be overwritten without explicit user confirmation. Distinguish "wipe and restart" from "abandon and stop".
- **Audit before commit.** The implementation is reviewed against the spec — not just self-reviewed — before it becomes a commit or PR.

## 2. The `.vibepilot/work/` layout

When a vibepilot spec is active, the workspace contains exactly four files. All four are always present (created as placeholders by `/vibepilot:new`) so commands can deterministically detect "empty" vs "substantive".

```
.vibepilot/work/
├── requirements.md    # WHY + WHAT (user-facing)         — written by /vibepilot:clarify
├── plan.md            # HOW (design + approach)          — written by /vibepilot:plan
├── tasks.md           # HOW, decomposed (checklist)      — written by /vibepilot:plan, flipped by /vibepilot:implement
└── review.md          # AUDIT (post-implementation)      — written by /vibepilot:review
```

### What "substantive" means

A file counts as substantive (and triggers append/rewrite gates instead of being treated as a placeholder) when it has **any `##` section beyond the `# Title`** — or, for `requirements.md` specifically, when it has a `> seed` line. Detection is structural, not size-based.

### File contracts

**`requirements.md`** — H1 `# Requirements`. Optional first body line `> <seed>` carried from `/vibepilot:new <seed>` is a hypothesis, not authoritative. Substantive content adds `##` sections like `## Background`, `## Goals`, `## Non-Goals`, `## Acceptance Criteria`. The exact set is fluid.

**`plan.md`** — H1 `# Plan`. `##` sections describe approach, architecture decisions, components touched. References `requirements.md` rather than restating it.

**`tasks.md`** — H1 `# Tasks`. Checklist lines match `^- \[[ x]\] (\d+)\. <title>$`, e.g. `- [ ] 3. wire the API client`. Numbering is contiguous from 1 and **append-only across rewrites** — new tasks start at `max_existing + 1`, even when old tasks are removed. Tasks may include a `Files:` sub-line listing source paths they cover, which `/vibepilot:review` uses to map tasks → diff. The set of `- [x]` lines is read by `/vibepilot:commit` (body bullets) and `/vibepilot:create-pr` (open-task count).

**`review.md`** — H1 `# Review`. First substantive line is a verdict: `PASS` / `ADVISORY` / `BLOCK`. `PASS` biases `/vibepilot:create-pr` toward Ready; anything else toward Draft. Sections detail per-criterion findings and deviations from plan.

## 3. When operating on these files

- **Don't fabricate sections.** Write only what the user (or the dedicated subagent for that file) supplies. Inventing `## Goals` to fill a placeholder breaks the "substantive vs empty" gate downstream commands rely on.
- **Don't renumber tasks**, and don't enumerate task titles into commit messages or PR bodies as a transcript. Numbering is structural state; readers of PRs and commits want feature behavior, not a step log.
- **Don't overwrite substantive content silently.** Use the rule above to detect, then ask the user — and distinguish *append/refine* from *rewrite from scratch*.
- **Keep `# Title` and `## Section` headers in canonical English** so parsers can find them, even when the body text is in the user's working language.

## When to apply this knowledge

- **Inside a `.vibepilot/work/` directory** → respect the file contracts (§2) and the operating rules (§3) before any read or edit.
- **The user asks "what does this checkbox mean", "what's the format of …", "why didn't `/vibepilot:plan` write tasks"** → §2 has the contract; the *why* is in §1.
- **The user is doing spec-driven dev in a project *without* `.vibepilot/work/`** → §1 still applies as general principles. Don't impose this file layout on them; let them keep their own conventions.
