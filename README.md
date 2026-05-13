# vibepilot

A minimal scaffold for building a [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin, distributed as a self-hosted single-plugin marketplace.

This repository ships **no** pre-built agents, commands, or skills. The directory layout, manifests, and install workflow are wired up — you add the content.

## Repository layout

```
vibepilot/
├── .claude-plugin/
│   └── marketplace.json          # marketplace metadata (this repo hosts itself)
├── plugins/
│   └── vibepilot/                # the plugin
│       ├── .claude-plugin/
│       │   └── plugin.json       # plugin manifest
│       ├── agents/               # subagent definitions (one .md per agent)
│       ├── commands/             # slash commands (one .md per command)
│       └── skills/               # skills (one subdir per skill, containing SKILL.md)
├── README.md
├── LICENSE
└── .gitignore
```

The three component directories are kept under git via `.gitkeep` placeholders. Delete the placeholder once you add real content.

## Install locally for development

```text
/plugin marketplace add /absolute/path/to/vibepilot
/plugin install vibepilot@vibepilot
```

After editing any file under `plugins/vibepilot/`, run `/plugin reload` in Claude Code to pick up the change without restarting.

Components you add will appear under the `vibepilot:` namespace — for example a command at `plugins/vibepilot/commands/hello.md` is invokable as `/vibepilot:hello`.

## Adding components

Each component is a markdown file with YAML frontmatter. Minimal shapes:

**Command** — `plugins/vibepilot/commands/<name>.md`

```markdown
---
description: One-line summary shown in the command palette
argument-hint: <optional hint, e.g. "<path>">
allowed-tools: Read, Grep, Bash
---

Instructions for the model when this command runs.
Use $ARGUMENTS to reference user-provided arguments.
```

**Agent** — `plugins/vibepilot/agents/<name>.md`

```markdown
---
name: <agent-name>
description: When and why to invoke this subagent
tools: Read, Grep, Glob, Bash
model: inherit
---

System prompt for the subagent.
```

**Skill** — `plugins/vibepilot/skills/<skill-name>/SKILL.md`

```markdown
---
name: <skill-name>
description: When this skill should auto-load into context
---

Skill content the model reads when the trigger matches.
```

For the authoritative reference on each component type (full frontmatter fields, loading rules, namespacing), see the [Claude Code plugin docs](https://docs.claude.com/en/docs/claude-code/plugins).

## Renaming for your own plugin

To fork this scaffold as a different plugin:

1. Edit `.claude-plugin/marketplace.json` — replace `name`, `plugins[0].name`, `plugins[0].description`, and `owner`.
2. Edit `plugins/vibepilot/.claude-plugin/plugin.json` — replace `name`, `description`, `author`, `homepage`, `repository`, `keywords`.
3. Rename the directory `plugins/vibepilot/` to match the new plugin name (the directory name and the `name` field in `plugin.json` should match).
4. Update this README.

## License

MIT. See [LICENSE](./LICENSE).
