# Marketplace Directory Structure

> How the marketplace repository is organized.

---

## Repository Layout

```
bailey-claude-marketplace/
├── .claude-plugin/
│   └── marketplace.json       # Marketplace catalog (required)
├── plugins/
│   ├── plugin-name-a/         # Each plugin in its own directory
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json    # Plugin manifest (required)
│   │   ├── skills/            # Skills and slash commands
│   │   ├── agents/            # Subagent definitions
│   │   ├── hooks/             # Event hooks
│   │   ├── .mcp.json          # MCP server config
│   │   └── scripts/           # Helper scripts
│   └── plugin-name-b/
│       └── ...
├── external_plugins/          # Plugins sourced from external repos (stubs only)
├── .trellis/                  # Project management (not distributed)
└── README.md
```

---

## Key Rules

1. **Marketplace root** must contain `.claude-plugin/marketplace.json`
2. **All plugins** live under `plugins/` — one directory per plugin
3. **Each plugin** must have `.claude-plugin/plugin.json` at its root
4. **Plugin names** use kebab-case: `my-cool-plugin`, not `myCoolPlugin`
5. **External plugins** (sourced from other repos via GitHub/git/npm) go in `external_plugins/` as lightweight stubs or are referenced purely in marketplace.json

---

## Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Plugin directory | kebab-case | `code-formatter` |
| Plugin name (in JSON) | kebab-case, matches directory | `"name": "code-formatter"` |
| Skill directory | kebab-case | `skills/quality-review/` |
| Command file | kebab-case `.md` | `commands/run-tests.md` |
| Agent file | kebab-case `.md` | `agents/security-reviewer.md` |
| Hook scripts | kebab-case | `scripts/validate-imports.sh` |

Name regex for plugins: `/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/`

---

## Reserved Names

These marketplace names are reserved by Anthropic and cannot be used:
- `claude-code-marketplace`, `claude-code-plugins`, `claude-plugins-official`
- `anthropic-marketplace`, `anthropic-plugins`
- `agent-skills`, `knowledge-work-plugins`, `life-sciences`
- Names that impersonate official marketplaces (e.g., `official-claude-plugins`)

---

## Anti-Patterns

- **Don't** put plugin files directly in the repo root — always under `plugins/<name>/`
- **Don't** use nested plugin directories (no `plugins/category/plugin-name/`)
- **Don't** reference files outside a plugin directory with `../` — plugins are copied to cache during install, so external references break
- **Don't** mix marketplace metadata with plugin code in the same directory
