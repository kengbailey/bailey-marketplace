# Quality Guidelines

> Validation, naming standards, and git workflow for the marketplace.

---

## Validation

### Before Committing

Run the built-in validator:

```bash
claude plugin validate .
```

Or from within Claude Code:

```
/plugin validate .
```

This checks:
- `marketplace.json` schema validity
- `plugin.json` schema per plugin
- Skill/agent/command frontmatter syntax
- `hooks/hooks.json` structure

### Common Validation Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Duplicate plugin name "x"` | Two plugins share the same name | Use unique names |
| `Path contains ".."` | Source path escapes plugin root | Use `./` relative paths only |
| `Plugin name is not kebab-case` | Uppercase, spaces, or special chars | Rename: `my-plugin-name` |
| `YAML frontmatter failed to parse` | Bad YAML in skill/command/agent | Fix YAML syntax |
| `Invalid JSON syntax` | Malformed JSON | Fix commas, quotes, brackets |

### Warnings (non-blocking but should fix)

- `Marketplace has no plugins defined` — add at least one plugin
- `No marketplace description provided` — add `metadata.description`
- `Plugin name is not kebab-case` — Claude Code accepts it but Claude.ai sync rejects it

---

## Naming Standards

| Item | Format | Pattern |
|------|--------|---------|
| Marketplace name | kebab-case | `/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/` |
| Plugin name | kebab-case | Same pattern |
| Skill directories | kebab-case | `quality-review`, not `qualityReview` |
| Agent files | kebab-case `.md` | `security-reviewer.md` |
| Command files | kebab-case `.md` | `run-tests.md` |
| Hook scripts | kebab-case | `validate-imports.sh` |

---

## JSON Formatting

- **2-space indentation** for all JSON files
- **Trailing newline** at end of file
- **No trailing commas** (strict JSON)
- **Sorted keys** in marketplace.json plugin entries (name, source, description, version, ...)

---

## Markdown Standards

- Skills/commands/agents use `.md` files with YAML frontmatter
- Frontmatter delimited by `---` on its own line
- Keep SKILL.md body between 1,500-2,000 words for optimal progressive disclosure
- Agent descriptions between 10-5,000 characters, including `<example>` blocks
- Command descriptions under 60 characters

---

## Git Workflow

### Commit Convention

```
type(scope): description
```

**Types**: `feat`, `fix`, `docs`, `refactor`, `chore`
**Scopes**: plugin name, `marketplace`, `docs`

Examples:
```
feat(quality-review): add quality review skill
fix(marketplace): correct source path for formatter plugin
docs(readme): add installation instructions
chore(marketplace): bump version for code-formatter
```

### Branch Strategy

- `main` — stable, validated marketplace
- `feat/<plugin-name>` — new plugin development
- `fix/<plugin-name>` — plugin fixes

### Pre-commit Checklist

1. `claude plugin validate .` passes
2. All plugin names are kebab-case
3. All plugin entries have `description`
4. External sources pinned with `ref` (and `sha` for production)
5. No duplicate plugin names

---

## Anti-Patterns

- **Don't** commit invalid JSON — always validate first
- **Don't** use inconsistent naming between directory name and plugin name in JSON
- **Don't** leave plugins without descriptions
- **Don't** reference unpinned external sources in production
