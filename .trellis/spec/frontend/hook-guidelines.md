# Plugin Hooks Guidelines

> Event-driven automation via hooks.json.

---

## Overview

Hooks let plugins react to Claude Code events. They live in `hooks/hooks.json` inside the plugin directory and use a **wrapper format** (different from settings.json hooks).

---

## hooks.json Format (Plugin Wrapper)

```json
{
  "description": "What these hooks do",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash ${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh",
            "timeout": 30
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Check if all tasks are complete before stopping.",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**Important:** Plugin hooks use the wrapper format with a top-level `"hooks"` key. Settings.json hooks do NOT use this wrapper.

---

## Hook Types

| Type | Description | Best For |
|------|-------------|----------|
| `command` | Runs a bash script | Deterministic validation, file checks |
| `prompt` | LLM-evaluated prompt | Context-aware decisions, subjective checks |

### Command Hook

```json
{
  "type": "command",
  "command": "bash ${CLAUDE_PLUGIN_ROOT}/scripts/check.sh",
  "timeout": 60
}
```

Default timeout: 60 seconds. Scripts receive JSON on stdin.

### Prompt Hook

```json
{
  "type": "prompt",
  "prompt": "Evaluate whether this tool use is appropriate for the task.",
  "timeout": 30
}
```

Default timeout: 30 seconds. Supported events: Stop, SubagentStop, UserPromptSubmit, PreToolUse.

---

## All 9 Hook Events

| Event | When | Common Use |
|-------|------|------------|
| `PreToolUse` | Before tool executes | Validate, approve/deny/modify tool calls |
| `PostToolUse` | After tool completes | Logging, feedback |
| `UserPromptSubmit` | User submits prompt | Context injection, validation |
| `Stop` | Main agent stopping | Completeness check |
| `SubagentStop` | Subagent stopping | Task validation |
| `SessionStart` | Session begins | Environment setup, context loading |
| `SessionEnd` | Session ends | Cleanup |
| `PreCompact` | Before context compaction | Preserve critical info |
| `Notification` | User notification fires | Logging |

---

## Matcher Patterns

| Pattern | Matches |
|---------|---------|
| `"Write"` | Exact tool name |
| `"Read\|Write\|Edit"` | Multiple tools (OR) |
| `".*"` or `"*"` | All tools |
| `"mcp__.*__delete.*"` | Regex pattern |

Matchers are **case-sensitive**.

---

## Hook Input (stdin JSON)

Command hooks receive this JSON on stdin:

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.txt",
  "cwd": "/current/working/dir",
  "permission_mode": "ask",
  "hook_event_name": "PreToolUse",
  "tool_name": "Write",
  "tool_input": { "file_path": "/path/to/file" },
  "tool_result": "..."
}
```

---

## Hook Output (stdout JSON)

### Standard Output

```json
{
  "continue": true,
  "suppressOutput": false,
  "systemMessage": "Optional message for Claude"
}
```

### PreToolUse-Specific

```json
{
  "hookSpecificOutput": {
    "permissionDecision": "allow",
    "updatedInput": { "field": "modified_value" }
  }
}
```

`permissionDecision`: `"allow"`, `"deny"`, or `"ask"`

### Stop/SubagentStop-Specific

```json
{
  "decision": "approve",
  "reason": "All tasks complete"
}
```

`decision`: `"approve"` or `"block"`

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success — stdout appears in transcript |
| 2 | Blocking error — stderr fed to Claude as context |
| Other | Non-blocking error — logged but ignored |

---

## SessionStart Special Behavior

SessionStart hooks can persist environment variables:

```bash
echo "export API_KEY=value" >> "$CLAUDE_ENV_FILE"
```

---

## Execution Model

- All matching hooks for an event run **in parallel**
- Hooks **cannot see** each other's output
- Hooks load at **session start** — changes require restart
- No hot-swapping

---

## Conventions for This Marketplace

1. **Prefer prompt hooks** for subjective decisions (Stop, PreToolUse)
2. **Use command hooks** for deterministic checks (file validation, syntax)
3. **Always use `${CLAUDE_PLUGIN_ROOT}`** in command paths
4. **Set reasonable timeouts** — don't exceed 60s for commands, 30s for prompts
5. **Use exit code 2** for errors that Claude should know about
6. **Document hooks** in the top-level `"description"` field

---

## Anti-Patterns

- **Don't** use the settings.json format in plugin hooks.json (missing wrapper causes load failure)
- **Don't** output invalid JSON from command hooks — causes silent failures
- **Don't** create hooks without matchers
- **Don't** rely on hook execution order — they run in parallel
- **Don't** hardcode paths — always use environment variables
