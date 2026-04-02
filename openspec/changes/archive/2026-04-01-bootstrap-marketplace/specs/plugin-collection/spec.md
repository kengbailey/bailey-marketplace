## ADDED Requirements

### Requirement: plugins directory exists
The repository SHALL contain a `plugins/` directory at the repo root for hosting in-repo plugins.

#### Scenario: Directory is present
- **WHEN** the repo is cloned
- **THEN** `plugins/` directory exists (preserved via `.gitkeep`)

### Requirement: README with install instructions
The repository SHALL contain a `README.md` at the repo root that documents how to add the marketplace and install plugins.

#### Scenario: README contains add command
- **WHEN** a user reads README.md
- **THEN** they find the `/plugin marketplace add` command with the correct repo path

#### Scenario: README contains install command
- **WHEN** a user reads README.md
- **THEN** they find the `/plugin install <plugin>@bailey-marketplace` command pattern

### Requirement: README documents plugin list
The README SHALL list all available plugins with their name, description, and source type (in-repo vs external).

#### Scenario: Plugin table is present
- **WHEN** a user reads README.md
- **THEN** they see a table or list of plugins with name, description, and whether it's in-repo or external
