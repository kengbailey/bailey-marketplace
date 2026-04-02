# JSON Schema Conventions

> Conventions for marketplace.json and plugin.json files.

---

## marketplace.json

Located at `.claude-plugin/marketplace.json` in the repository root.

### Required Fields

```json
{
  "name": "bailey-marketplace",
  "owner": {
    "name": "Team Name",
    "email": "optional@example.com"
  },
  "plugins": [
    {
      "name": "plugin-name",
      "source": "./plugins/plugin-name",
      "description": "Brief description of what this plugin does"
    }
  ]
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | Yes | Marketplace identifier, kebab-case |
| `owner.name` | string | Yes | Maintainer name |
| `owner.email` | string | No | Contact email |
| `plugins` | array | Yes | List of plugin entries |

### Optional Metadata

```json
{
  "metadata": {
    "description": "Brief marketplace description",
    "version": "1.0.0",
    "pluginRoot": "./plugins"
  }
}
```

`pluginRoot` is prepended to relative source paths, so `"source": "formatter"` resolves to `./plugins/formatter`.

---

## Plugin Entries in marketplace.json

Each plugin entry supports these fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | Yes | Kebab-case identifier |
| `source` | string or object | Yes | Where to fetch the plugin |
| `description` | string | Yes* | Required by our convention |
| `version` | string | No | Semver (set here OR in plugin.json, not both) |
| `category` | string | No | For organization |
| `tags` | array | No | For searchability |
| `author` | object | No | `{name, email?}` |
| `homepage` | string | No | Documentation URL |
| `license` | string | No | SPDX identifier |
| `strict` | boolean | No | Default: true. See below. |

### Source Types

**Relative path** (for plugins in this repo):
```json
{ "source": "./plugins/my-plugin" }
```

**GitHub repo**:
```json
{
  "source": {
    "source": "github",
    "repo": "owner/repo",
    "ref": "v1.0.0",
    "sha": "a1b2c3d4..."
  }
}
```

**Git URL** (GitLab, Bitbucket, etc.):
```json
{
  "source": {
    "source": "url",
    "url": "https://gitlab.com/team/plugin.git",
    "ref": "main"
  }
}
```

**Git subdirectory** (monorepo):
```json
{
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/org/monorepo.git",
    "path": "tools/claude-plugin"
  }
}
```

**npm package**:
```json
{
  "source": {
    "source": "npm",
    "package": "@scope/plugin-name",
    "version": "^2.0.0"
  }
}
```

---

## plugin.json

Located at `<plugin-root>/.claude-plugin/plugin.json`.

### Minimal (required)

```json
{
  "name": "my-plugin"
}
```

### Standard

```json
{
  "name": "my-plugin",
  "description": "What this plugin does in 50-200 characters",
  "version": "1.0.0",
  "author": {
    "name": "Author Name"
  },
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"]
}
```

### Component Path Overrides (optional)

```json
{
  "name": "my-plugin",
  "commands": ["./commands", "./extra-commands"],
  "agents": ["./agents"],
  "hooks": "./hooks/hooks.json",
  "mcpServers": "./.mcp.json"
}
```

Default directories (`./commands`, `./agents`, `./skills`) are always scanned. Custom paths **supplement** defaults, they don't replace them.

---

## Strict Mode

| Value | Behavior |
|-------|----------|
| `true` (default) | plugin.json is the authority. Marketplace entry can add extra components (merged). |
| `false` | Marketplace entry is the entire definition. If plugin.json also declares components, it fails to load. |

Use `strict: true` for most plugins. Use `strict: false` when the marketplace needs full control over a plugin's components.

---

## Conventions for This Marketplace

1. **Always include `description`** in plugin entries
2. **Set version in one place only** — marketplace.json for in-repo plugins, plugin.json for external
3. **Use `category` and `tags`** for all plugins to aid discovery
4. **Pin external sources** with `ref` and `sha` for reproducibility
5. **2-space indentation** for all JSON files
6. **No trailing commas** (strict JSON)

---

## Anti-Patterns

- **Don't** set version in both marketplace.json and plugin.json — creates silent conflicts
- **Don't** use `../` in any source paths
- **Don't** use uppercase or spaces in names — strictly kebab-case
- **Don't** leave `description` empty on plugin entries
