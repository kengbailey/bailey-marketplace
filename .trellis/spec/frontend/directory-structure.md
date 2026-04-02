# Plugin Directory Structure

> Internal structure of each plugin in the marketplace.

---

## Standard Plugin Layout

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Required: plugin manifest
├── skills/                   # Preferred: skills and slash commands
│   └── skill-name/
│       ├── SKILL.md          # Required per skill
│       ├── references/       # Detailed docs (loaded on demand)
│       ├── examples/         # Code samples
│       ├── scripts/          # Executable utilities
│       └── assets/           # Templates, images
├── agents/                   # Subagent definitions
│   └── agent-name.md
├── commands/                 # Legacy slash commands (still works)
│   └── command-name.md
├── hooks/
│   └── hooks.json            # Event hook configuration
├── .mcp.json                 # MCP server definitions
├── scripts/                  # Shared helper scripts
└── README.md                 # Plugin documentation
```

---

## Key Rules

1. **`.claude-plugin/plugin.json`** is the only required file
2. **Component directories are optional** — only create what the plugin uses
3. **`skills/` is preferred** over `commands/` for slash commands (both work identically)
4. **Each skill gets its own subdirectory** with `SKILL.md` at the root
5. **All paths in configs** must use `./` prefix and forward slashes
6. **No `../`** — plugins are copied to cache, external references break

---

## Progressive Disclosure (Skills)

Skills use a three-level loading strategy:

| Level | What | When Loaded | Size Guidance |
|-------|------|-------------|---------------|
| 1. Metadata | name + description from frontmatter | Always in context | ~100 words |
| 2. Body | SKILL.md content | When skill triggers | 1,500-2,000 words |
| 3. Resources | `references/`, `examples/`, `scripts/` | On demand via `@file` | Unlimited |

Structure skill directories to take advantage of this:
- Put the essential instructions in SKILL.md body
- Put detailed references, schemas, and examples in subdirectories
- Use `@${CLAUDE_PLUGIN_ROOT}/skills/name/references/detail.md` to load on demand

---

## Minimal Plugin Example

```
hello-world/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── hello/
        └── SKILL.md
```

```json
// .claude-plugin/plugin.json
{
  "name": "hello-world"
}
```

```markdown
<!-- skills/hello/SKILL.md -->
---
description: Say hello when asked
---

Respond with a friendly greeting.
```

---

## Standard Plugin Example

```
code-quality/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── quality-review/
│       ├── SKILL.md
│       └── references/
│           └── checklist.md
├── agents/
│   └── security-reviewer.md
├── hooks/
│   └── hooks.json
└── scripts/
    └── validate-imports.sh
```

---

## Environment Variables in Plugins

| Variable | Available In | Description |
|----------|-------------|-------------|
| `${CLAUDE_PLUGIN_ROOT}` | hooks, MCP configs, commands | Absolute path to plugin install directory |
| `${CLAUDE_PLUGIN_DATA}` | hooks, scripts | Persistent data directory (survives updates) |
| `${CLAUDE_PROJECT_DIR}` | hooks, scripts | User's project root |
| `${CLAUDE_ENV_FILE}` | SessionStart hooks only | Path to write env vars for persistence |

**Always use `${CLAUDE_PLUGIN_ROOT}`** instead of hardcoded paths — plugins are copied to `~/.claude/plugins/cache/` when installed.

---

## Anti-Patterns

- **Don't** reference files outside the plugin directory — they won't be copied
- **Don't** hardcode absolute paths — use `${CLAUDE_PLUGIN_ROOT}`
- **Don't** put plugin.json at the plugin root — it goes in `.claude-plugin/`
- **Don't** nest skills deeper than `skills/<name>/SKILL.md`
- **Don't** mix component types in one directory (e.g., agents in the skills folder)
