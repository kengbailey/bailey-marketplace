## Why

This repo has no marketplace structure yet — it's an empty project with only development guidelines. We need to scaffold the initial `.claude-plugin/marketplace.json` catalog and add the first external plugin (claude-mem) so the marketplace is installable and functional via `/plugin marketplace add`.

## What Changes

- Create `.claude-plugin/marketplace.json` with marketplace metadata and claude-mem as the first plugin entry
- Create `plugins/` directory structure for future in-repo plugins
- Create `README.md` with installation and usage instructions
- Add claude-mem as an external plugin reference (unpinned GitHub source)

## Capabilities

### New Capabilities

- `marketplace-catalog`: The core marketplace.json catalog that defines the marketplace identity, owner, and lists all plugins with their sources
- `plugin-collection`: The directory structure and conventions for hosting in-repo plugins and referencing external plugins

### Modified Capabilities

(none — greenfield project)

## Impact

- **New files**: `.claude-plugin/marketplace.json`, `plugins/.gitkeep`, `README.md`
- **Dependencies**: claude-mem is referenced as an external GitHub source (`thedotmack/claude-mem`) — no vendored code
- **Users**: After this change, anyone can install the marketplace with `/plugin marketplace add <owner>/bailey-claude-marketplace`
