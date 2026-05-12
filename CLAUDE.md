# dotclaude — Claude Code Plugin Repo

This repo is a personal Claude Code **marketplace** containing the `personal` plugin with skills, hooks, and MCP server configs.

## Repo structure

```
dotclaude/
├── .claude-plugin/marketplace.json   # marketplace manifest
└── personal/                         # the "personal" plugin
    ├── .claude-plugin/plugin.json    # plugin manifest
    ├── hooks.json                    # lifecycle hooks
    ├── mcp.json                      # MCP server configs
    └── skills/
        └── <skill-name>/
            └── SKILL.md              # one skill per directory
```

## Version bumping (mandatory on every change)

The `version` field in `plugin.json` is the cache key. If it doesn't change, Claude Code serves the cached copy — skill edits are invisible to users.

**After every commit that modifies skills, hooks, or MCP config, bump both files:**

| File | Field |
|---|---|
| `personal/.claude-plugin/plugin.json` | `"version"` |
| `.claude-plugin/marketplace.json` | `"version"` |

**Semver convention:**
- Patch (`1.0.x`) — skill text edits, bug fixes, description tweaks
- Minor (`1.x.0`) — new skill, new hook, new MCP server, backward-compatible additions
- Major (`x.0.0`) — removing or renaming a skill, breaking hook or MCP changes

> Alternative: omit `version` entirely to use git commit SHA as the version. Every push then delivers updates automatically. Useful if you prefer not to manage versions manually.

## Adding a new skill

1. Create `personal/skills/<skill-name>/SKILL.md`
2. Write the frontmatter and instructions (see template below)
3. Bump the minor version in both manifest files
4. Commit and push — start a new Claude Code session to load the skill

**SKILL.md template:**

```markdown
---
name: skill-name
description: >
  One or two sentences. What the skill does and exactly when Claude should
  invoke it. This is what Claude reads to decide auto-invocation — be specific.
  Include example trigger phrases ("Use when the user says X, Y, Z").
allowed-tools: Bash, Read, Edit, Write, AskUserQuestion
---

## Your instructions here

Step-by-step instructions for Claude to follow when this skill is invoked.
```

## Skill authoring best practices

- **Description is the trigger** — Claude uses it for automatic invocation. Include concrete trigger phrases. Keep it under 1,536 chars total.
- **Keep skills under 500 lines** — move reference material to supporting files and `@import` them.
- **`allowed-tools` pre-approves, does not restrict** — listed tools skip permission prompts; unlisted tools are still accessible.
- **Use `disable-model-invocation: true`** for skills with side effects (deploy, commit, send message) to require explicit user invocation.
- **Dynamic context** — prefix a line with `` !`command` `` to inject shell output before Claude sees the skill (e.g., `` !`git status` ``).
- **CLAUDE.md is not loaded for plugin users** — it is only for contributors working on this repo. Plugin context must live in skills.

## Plugin manifest fields used here

`personal/.claude-plugin/plugin.json` — only `name`, `version`, `hooks`, and `mcpServers` are set. Skills are auto-discovered from `personal/skills/`.

Do not move `skills/`, `hooks.json`, or `mcp.json` inside `.claude-plugin/` — only `plugin.json` lives there.
