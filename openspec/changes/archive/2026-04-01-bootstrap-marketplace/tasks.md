## 1. Marketplace Structure

- [x] 1.1 Create `.claude-plugin/` directory at repo root
- [x] 1.2 Create `.claude-plugin/marketplace.json` with marketplace name (`bailey-marketplace`), owner, and empty plugins array
- [x] 1.3 Add claude-mem plugin entry to marketplace.json with name, description, category, and unpinned GitHub source (`thedotmack/claude-mem`)

## 2. Plugin Directory

- [x] 2.1 Create `plugins/` directory with `.gitkeep` file

## 3. Documentation

- [x] 3.1 Create `README.md` with marketplace description, installation command (`/plugin marketplace add`), and plugin install command (`/plugin install`)
- [x] 3.2 Add plugin table to README listing available plugins (name, description, source type)
- [x] 3.3 Document claude-mem's dependencies and setup notes in README

## 4. Validation

- [x] 4.1 Run `claude plugin validate .` and confirm no errors
