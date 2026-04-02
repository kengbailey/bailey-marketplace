## ADDED Requirements

### Requirement: Valid marketplace.json at repo root
The repository SHALL contain a `.claude-plugin/marketplace.json` file that conforms to the Claude Code marketplace schema and passes `claude plugin validate .`.

#### Scenario: Marketplace validation passes
- **WHEN** `claude plugin validate .` is run from the repo root
- **THEN** validation passes with no errors

#### Scenario: Marketplace is installable
- **WHEN** a user runs `/plugin marketplace add <owner>/bailey-claude-marketplace`
- **THEN** the marketplace is added successfully and its plugins are listed

### Requirement: Marketplace identity
The marketplace.json SHALL define `name` as `bailey-marketplace` and include an `owner` object with at minimum a `name` field.

#### Scenario: Marketplace name is set
- **WHEN** marketplace.json is parsed
- **THEN** `name` equals `bailey-marketplace` and `owner.name` is defined

### Requirement: claude-mem external plugin entry
The marketplace.json `plugins` array SHALL contain an entry for claude-mem with `name`, `description`, `source` (GitHub type pointing to `thedotmack/claude-mem`), and `category`.

#### Scenario: claude-mem is listed
- **WHEN** a user runs `/plugin marketplace add` and then lists available plugins
- **THEN** `claude-mem` appears in the plugin list with its description

#### Scenario: claude-mem is installable
- **WHEN** a user runs `/plugin install claude-mem@bailey-marketplace`
- **THEN** the plugin is fetched from `thedotmack/claude-mem` on GitHub and installed

### Requirement: Plugin entries include description
Every plugin entry in the marketplace.json `plugins` array SHALL include a non-empty `description` field.

#### Scenario: No plugin without description
- **WHEN** marketplace.json is parsed
- **THEN** every object in the `plugins` array has a `description` string with length > 0
