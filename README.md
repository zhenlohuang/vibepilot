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

Ten slash commands — six that drive a five-step spec-driven workflow (plus `clean`) and four that wrap the surrounding git / GitHub PR loop — plus four internal subagents that handle the heavy lifting (clarification, codebase exploration, code edits, review) so your main conversation stays uncluttered.

All spec artifacts live in `.vibepilot/work/` in your project. Only **one** spec lives there at a time — vibepilot is opinionated about this on purpose, so you always know which task is in flight. The directory persists across sessions, so you can close Claude Code mid-task and resume by running the next command.

> Add `.vibepilot/` to your `.gitignore` (or commit it deliberately if you want spec history under version control — that's a team-policy call).

## Commands

| command | dependency | description |
|---|---|---|
| `/vibepilot:new [description]` | — | Initialize a fresh spec workspace at `.vibepilot/work/`. If a spec is already in progress, asks to clear or cancel before overwriting. |
| `/vibepilot:clarify` | — | Interactively capture requirements (goal, user stories, acceptance criteria, constraints, non-goals). Writes `requirements.md`. |
| `/vibepilot:plan` | — | Design the implementation and decompose it into an ordered checkbox list via the `vibe-planner` subagent. Writes `plan.md` + `tasks.md`. |
| `/vibepilot:implement [scope]` | — | Execute remaining tasks via the `vibe-developer` subagent; edits source code and ticks `- [x]` in `tasks.md`. Defaults to every `- [ ]` in order; pass any scope hint to narrow. Stops on the first hard blocker. |
| `/vibepilot:review [scope]` | — | Audit the spec implementation via the `vibe-reviewer` subagent. Defaults to uncommitted changes (or branch-vs-base if the tree is clean); pass any scope hint — task references, a branch name, file paths — to narrow or shift focus. Writes `review.md`. |
| `/vibepilot:commit` | `git` | Stage and commit the current changes with a message synthesized from the spec (or the staged diff if no spec is active). Offers to push. |
| `/vibepilot:create-pr` | `git`, `gh` | Open a pull request for the current branch. Title and body come from the spec when one is active, otherwise from `git log <base>..HEAD`. |
| `/vibepilot:review-pr [PR\|URL]` | `gh` | Review an existing GitHub PR (thin wrapper around Claude Code's built-in `/review`). |
| `/vibepilot:merge-pr [PR\|URL]` | `gh` | Merge a GitHub PR with the repo's allowed merge method, after the mergeability gates pass. |
| `/vibepilot:clean` | — | Remove `.vibepilot/work/` entirely (with confirmation). No placeholders left behind. |

### Typical flow

```text
/vibepilot:new        add a CSV export to the reports page
/vibepilot:clarify    # answer questions to lock down requirements
/vibepilot:plan       # planner subagent drafts the approach + ordered task checklist
/vibepilot:implement  # developer subagent codes tasks 1..N
/vibepilot:review     # reviewer subagent audits against the spec
```

Both `implement` and `review` default to a sensible no-arg behavior — `implement` walks every remaining `- [ ]` task in order, `review` audits whatever's uncommitted (falling back to branch-vs-base if the tree is clean) — so you can iterate `implement` → `review` → `commit` without typing anything. Describe a scope in whatever shape feels natural (a number, a range, a branch, a file path, a free-form sentence) when you want to narrow or shift focus.

### Switching tasks

`/vibepilot:new` won't overwrite an in-progress spec silently — it surfaces the in-flight state and asks `Clear and start fresh` vs `Cancel`. To wipe without re-initializing, run `/vibepilot:clean` instead.

### Resuming

Closed your terminal halfway through? Just open Claude Code in the same project and run the next command — `.vibepilot/work/` is the source of truth, and every command picks up wherever the files left off.

## Repository layout

```
vibepilot/
├── .claude-plugin/marketplace.json   # marketplace metadata (self-hosted)
└── plugins/vibepilot/
    ├── .claude-plugin/plugin.json    # plugin manifest
    ├── agents/                       # vibe-clarifier, vibe-planner, vibe-developer, vibe-reviewer
    └── skills/                       # new, clarify, plan, implement, review, commit, create-pr, review-pr, merge-pr, clean
```

## License

MIT. See [LICENSE](./LICENSE).
