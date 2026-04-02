# Skills & Commands Guidelines

> How to author skills (slash commands), agents, and legacy commands.

---

## Skills (Preferred Format)

Skills are the preferred way to add capabilities to plugins. They live in `skills/<name>/SKILL.md`.

### Two Types

| Type | Activation | Key Frontmatter |
|------|-----------|-----------------|
| **Model-invoked** | Automatically when task matches description | `name`, `description` |
| **User-invoked** (slash command) | User types `/skill-name` | `name`, `description`, `argument-hint`, `allowed-tools`, `model` |

### SKILL.md Frontmatter

```yaml
---
name: quality-review
description: Review code for bugs, security, and performance issues
version: 1.0.0
argument-hint: "[file-path]"
allowed-tools:
  - Read
  - Grep
  - Glob
model: sonnet
disable-model-invocation: true
license: MIT
---
```

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | Skill identifier |
| `description` | Yes | Third-person for model-invoked. Include trigger phrases. |
| `version` | No | Semver |
| `argument-hint` | No | For user-invoked: shown in autocomplete |
| `allowed-tools` | No | Tool restrictions for user-invoked skills |
| `model` | No | `sonnet`, `opus`, `haiku` |
| `disable-model-invocation` | No | Set `true` for user-only slash commands |
| `license` | No | SPDX identifier |

### SKILL.md Body

The body is the instruction set loaded when the skill activates. Write in **imperative form**:

```markdown
---
description: This skill reviews code for quality issues
disable-model-invocation: true
---

Review the code I've selected or the recent changes for:
- Potential bugs or edge cases
- Security concerns
- Performance issues
- Readability improvements

Be concise and actionable.
```

**Writing rules:**
- Imperative/infinitive form (not "you should")
- Description in third person ("This skill should be used when...")
- 1,500-2,000 words optimal for body
- Use `@${CLAUDE_PLUGIN_ROOT}/references/detail.md` for extended content

### Dynamic Features in Skills

```markdown
<!-- User arguments -->
Process $ARGUMENTS and apply to the current context.
The first argument is $1, second is $2.

<!-- File references -->
Follow the patterns in @${CLAUDE_PLUGIN_ROOT}/references/patterns.md

<!-- Inline bash for context -->
Current git status: !`git status --short`
```

---

## Agents

Agent definitions live in `agents/<name>.md`. They define specialized subagents Claude can delegate to.

### Agent Frontmatter

```yaml
---
name: security-reviewer
description: |
  Use this agent when reviewing code for security vulnerabilities.

  <example>
  Context: User asks to check their authentication code
  user: "Review the auth module for security issues"
  assistant: "I'll delegate this to the security reviewer agent"
  <commentary>
  The security reviewer specializes in finding auth vulnerabilities
  </commentary>
  </example>
model: inherit
color: red
tools:
  - Read
  - Grep
  - Glob
---
```

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | 3-50 chars, kebab-case |
| `description` | Yes | 10-5,000 chars. MUST include `<example>` blocks (2-4) |
| `model` | Yes | `inherit` (recommended), `sonnet`, `opus`, `haiku` |
| `color` | Yes | `blue`, `cyan`, `green`, `yellow`, `magenta`, `red` |
| `tools` | No | Tool allowlist. Default: all tools. |

### Agent Body

The body becomes the agent's system prompt. Write in **second person** (addressing the agent):

```markdown
You are a security review specialist. When reviewing code:

1. Check for OWASP Top 10 vulnerabilities
2. Verify input validation at all boundaries
3. Check for hardcoded secrets or credentials
4. Review authentication and authorization logic

Report findings in a structured format with severity levels.
```

**Sizing:** System prompt body 500-3,000 characters works best.

---

## Legacy Commands

Commands in `commands/<name>.md` still work but `skills/` is preferred. They use the same frontmatter fields as user-invoked skills.

```markdown
---
description: Run the test suite
allowed-tools:
  - Bash
---

Run all tests and report results.
```

Subdirectories create namespaces: `commands/ci/build.md` registers as a namespaced command.

---

## Conventions for This Marketplace

1. **Use `skills/` format** for all new slash commands (not `commands/`)
2. **Always set `disable-model-invocation: true`** for user-only commands
3. **Include 2-4 `<example>` blocks** in agent descriptions
4. **Keep descriptions under 60 characters** for autocomplete readability
5. **Use `model: inherit`** for agents unless a specific model is needed
6. **Reference files** with `@${CLAUDE_PLUGIN_ROOT}/...` for portability

---

## Anti-Patterns

- **Don't** write skills in second person ("you should...") — use imperative form
- **Don't** write agent descriptions without examples — they won't trigger correctly
- **Don't** create agents with system prompts over 5,000 characters
- **Don't** use absolute paths in file references
- **Don't** put skill resources directly in the skill directory — use `references/` and `examples/` subdirectories
