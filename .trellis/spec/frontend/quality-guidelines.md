# Plugin Quality Guidelines

> Standards for testing and validating plugins before distribution.

---

## Validation

### Validate Plugin Structure

```bash
claude plugin validate .
# or from within Claude Code:
/plugin validate .
```

### Test Plugin Locally

```bash
# Install from local directory for testing
cc --plugin-dir /path/to/plugin

# Enable debug logging
claude --debug
```

### Debug Commands

| Command | Purpose |
|---------|---------|
| `/hooks` | Review loaded hooks |
| `/mcp` | View MCP servers and tools |
| `/help` | See all registered commands/skills |

---

## Plugin Checklist

### Required

- [ ] `.claude-plugin/plugin.json` exists with valid `name`
- [ ] `claude plugin validate .` passes with no errors
- [ ] Plugin name matches directory name
- [ ] All file references use `${CLAUDE_PLUGIN_ROOT}` (not absolute paths)
- [ ] No files reference outside the plugin directory (`../`)

### Recommended

- [ ] `description` is 50-200 characters
- [ ] `version` follows semver
- [ ] `keywords` included (5-10 for discovery)
- [ ] README.md documents what the plugin does
- [ ] Required environment variables documented

### Skills Quality

- [ ] SKILL.md body is 1,500-2,000 words
- [ ] Description is in third person for model-invoked skills
- [ ] `disable-model-invocation: true` set for user-only commands
- [ ] References organized in `references/` subdirectory

### Agent Quality

- [ ] Description includes 2-4 `<example>` blocks
- [ ] System prompt body is 500-3,000 characters
- [ ] `model: inherit` unless specific model needed
- [ ] `color` is set

### Hook Quality

- [ ] hooks.json uses wrapper format (top-level `"hooks"` key)
- [ ] All command paths use `${CLAUDE_PLUGIN_ROOT}`
- [ ] Matchers are specific (not all `".*"` unless intentional)
- [ ] Timeouts are reasonable (command: <=60s, prompt: <=30s)
- [ ] Command scripts handle invalid stdin JSON gracefully

### MCP Server Quality

- [ ] No hardcoded secrets in `.mcp.json`
- [ ] Required env vars documented in README
- [ ] Server starts reliably (test with `/mcp`)

---

## Testing Hooks

Test command hooks manually:

```bash
echo '{"tool_name": "Write", "tool_input": {"file_path": "/test"}}' | \
  bash ${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh
```

Verify output is valid JSON with expected fields.

---

## Review Checklist (Before Adding to Marketplace)

1. **Validate**: `claude plugin validate .` passes
2. **Test**: Install locally and verify all components work
3. **Document**: README explains purpose, setup, and usage
4. **Naming**: All names are kebab-case and descriptive
5. **Portability**: No hardcoded paths, all env vars documented
6. **Security**: No secrets in committed files

---

## Anti-Patterns

- **Don't** distribute plugins that fail validation
- **Don't** skip local testing before adding to marketplace
- **Don't** leave placeholder/template content in published skills
- **Don't** create plugins without documentation
