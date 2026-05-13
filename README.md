# vibepilot

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin that turns "vibe coding" into a structured, spec-driven workflow. You stay in flow; vibepilot keeps the spec, plan, tasks, and review tidy in the background.

Distributed as a self-hosted single-plugin marketplace — install straight from this repo, no registry required.

## Installation

Inside Claude Code, add this repo as a marketplace and install the plugin:

```text
/plugin marketplace add zhenlohuang/vibepilot
/plugin install vibepilot@vibepilot
```

Or, if you've cloned the repo locally and want to hack on it:

```text
/plugin marketplace add /absolute/path/to/vibepilot
/plugin install vibepilot@vibepilot
```

After install, all commands are namespaced under `vibepilot:` — e.g. `/vibepilot:new`. When developing locally, run `/plugin reload` to pick up edits without restarting Claude Code.

## What you get

Six slash commands that drive a five-step spec-driven workflow, plus four internal subagents that handle the heavy lifting (clarification, codebase exploration, code edits, review) so your main conversation stays uncluttered.

All spec artifacts live in `.vibepilot/spec/` in your project. Only **one** spec lives there at a time — vibepilot is opinionated about this on purpose, so you always know which task is in flight. The directory persists across sessions, so you can close Claude Code mid-task and resume by running the next command.

> Add `.vibepilot/` to your `.gitignore` (or commit it deliberately if you want spec history under version control — that's a team-policy call).

## Commands

| Command | What it does | Writes to |
|---|---|---|
| `/vibepilot:new [description]` | Initialize a fresh spec workspace. Refuses if a spec is already in progress. | `.vibepilot/spec/*.md` (placeholders) |
| `/vibepilot:clarify` | Interactively capture requirements (goal, user stories, acceptance criteria, constraints, non-goals). | `.vibepilot/spec/requirements.md` |
| `/vibepilot:plan` | Design the implementation and decompose it into an ordered checkbox list. Delegates to the `vibe-planner` subagent. | `.vibepilot/spec/plan.md` + `.vibepilot/spec/tasks.md` |
| `/vibepilot:implement [N\|N-M\|all]` | Execute tasks via the `vibe-developer` subagent; ticks completed checkboxes. | source code + `tasks.md` |
| `/vibepilot:review [N\|N-M\|all]` | Audit the changes against the spec via the `vibe-reviewer` subagent. | `.vibepilot/spec/review.md` |
| `/vibepilot:clean` | Remove `.vibepilot/spec/` entirely (with confirmation). No placeholders left behind. | deletes `.vibepilot/spec/` |

### Typical flow

```text
/vibepilot:new        add a CSV export to the reports page
/vibepilot:clarify    # answer questions to lock down requirements
/vibepilot:plan       # planner subagent drafts the approach + ordered task checklist
/vibepilot:implement  # developer subagent codes tasks 1..N
/vibepilot:review     # reviewer subagent audits against the spec
```

You can run partial passes — e.g. `/vibepilot:implement 2-4` to do just tasks 2 through 4, then `/vibepilot:review 2-4` to audit only those.

### Switching tasks

`/vibepilot:new` is non-destructive: it won't overwrite an in-progress spec. To start a different task, run `/vibepilot:clean` first (you'll be asked to confirm), then `/vibepilot:new` again.

### Resuming

Closed your terminal halfway through? Just open Claude Code in the same project and run the next command — `.vibepilot/spec/` is the source of truth, and every command picks up wherever the files left off.

## Repository layout

```
vibepilot/
├── .claude-plugin/marketplace.json   # marketplace metadata (self-hosted)
└── plugins/vibepilot/
    ├── .claude-plugin/plugin.json    # plugin manifest
    ├── agents/                       # vibe-planner, vibe-developer, vibe-reviewer
    └── skills/                       # new, clarify, plan, implement, review, clean
```

## License

MIT. See [LICENSE](./LICENSE).
