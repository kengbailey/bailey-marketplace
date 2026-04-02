## Context

This is a greenfield Claude Code plugin marketplace repo. Currently it contains only `.trellis/` project management scaffolding and `openspec/` change tracking. No marketplace structure, no plugins, no README.

The Claude Code plugin system expects a specific structure: `.claude-plugin/marketplace.json` at repo root, with plugins either co-located under `plugins/` or referenced externally via GitHub/git/npm sources.

## Goals / Non-Goals

**Goals:**
- Scaffold a valid, installable marketplace that passes `claude plugin validate .`
- Add claude-mem as the first external plugin entry
- Establish the directory structure for future in-repo plugins
- Provide a README so users know how to install and use the marketplace

**Non-Goals:**
- Building any in-repo plugins (future work)
- Version pinning external plugins (Option A: unpinned for now)
- Adding categories, tags, or rich metadata beyond basics (can evolve later)
- CI/CD, automated validation, or release workflows

## Decisions

### 1. Marketplace name: `bailey-marketplace`

Personal marketplace, simple and recognizable. Avoids reserved names (claude-*, anthropic-*). Users install plugins as `plugin-name@bailey-marketplace`.

### 2. External plugin sourcing: unpinned GitHub reference

```json
{ "source": "github", "repo": "thedotmack/claude-mem" }
```

**Why not pinned?** This is a personal marketplace — we want the latest version automatically. Pinning with `sha` is better for team/production marketplaces where reproducibility matters. We can add pinning later if stability becomes a concern.

**Alternative considered:** `git-subdir` pointing to `thedotmack/claude-mem` with `path: "plugin"` (since claude-mem's actual plugin lives in a `plugin/` subdirectory of its repo). However, the repo's own marketplace.json already handles this mapping, and the GitHub source type clones the full repo which includes the plugin discovery logic.

### 3. Directory structure: `plugins/` with .gitkeep

A `plugins/` directory with `.gitkeep` signals where in-repo plugins will live. This matches the convention used by other community marketplaces and aligns with the `pluginRoot` metadata option.

### 4. No `metadata.pluginRoot` for now

We'll use explicit `./plugins/<name>` paths in source fields rather than setting a global `pluginRoot`. This is more readable when the marketplace is small and avoids confusion about path resolution.

## Risks / Trade-offs

- **[Unpinned external source]** claude-mem could push a breaking change that affects users → We accept this for a personal marketplace; can pin later if needed
- **[claude-mem complexity]** It has heavy dependencies (Bun, uv, Chroma, tree-sitter) that auto-install on first run → Users should be aware; document in README
- **[Empty plugins/ directory]** May look incomplete → .gitkeep and README explain this is intentional
