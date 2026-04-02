# MCP Server Guidelines

> Configuring and distributing MCP servers in plugins.

---

## Overview

MCP (Model Context Protocol) servers let plugins expose tools, resources, and prompts to Claude. Configuration lives in `.mcp.json` at the plugin root.

---

## .mcp.json Format

```json
{
  "mcpServers": {
    "server-name": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/my-server.js"],
      "env": {
        "API_KEY": "${API_KEY}"
      }
    }
  }
}
```

---

## Server Types

### stdio (default, most common)

Local process communicating via stdin/stdout. No `type` field needed.

```json
{
  "mcpServers": {
    "my-tool": {
      "command": "npx",
      "args": ["-y", "@scope/mcp-server"],
      "env": { "TOKEN": "${TOKEN}" }
    }
  }
}
```

### SSE (Server-Sent Events)

For hosted/cloud services:

```json
{
  "mcpServers": {
    "cloud-tool": {
      "type": "sse",
      "url": "https://mcp.example.com/sse"
    }
  }
}
```

### HTTP

REST-based with optional OAuth:

```json
{
  "mcpServers": {
    "api-tool": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${TOKEN}"
      }
    }
  }
}
```

With OAuth:

```json
{
  "mcpServers": {
    "oauth-tool": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "oauth": {
        "clientId": "your-client-id",
        "callbackPort": 3118
      }
    }
  }
}
```

### WebSocket

For real-time streaming:

```json
{
  "mcpServers": {
    "ws-tool": {
      "type": "ws",
      "url": "wss://stream.example.com/mcp",
      "headers": { "Authorization": "Bearer ${TOKEN}" }
    }
  }
}
```

---

## Tool Naming Convention

MCP tools from plugins are registered as:

```
mcp__plugin_<plugin-name>_<server-name>__<tool-name>
```

Example: plugin `code-tools` with server `formatter` exposing tool `format-file`:
```
mcp__plugin_code-tools_formatter__format-file
```

---

## Environment Variable Expansion

All MCP configs support variable expansion:

| Variable | Source |
|----------|--------|
| `${CLAUDE_PLUGIN_ROOT}` | Plugin install directory |
| `${API_KEY}` | User's shell environment |
| `${HOME}` | User's home directory |

**Secrets should never be hardcoded** — always reference environment variables.

---

## Lifecycle

1. MCP servers **start when plugin enables**
2. Connections are **lazy** (on-demand)
3. Config changes require **Claude Code restart**
4. Check server status with `/mcp` command

---

## Conventions for This Marketplace

1. **Prefer stdio** for local tools (simplest, most portable)
2. **Use `${CLAUDE_PLUGIN_ROOT}`** for all file paths in server configs
3. **Document required env vars** in plugin README and SKILL.md
4. **Never hardcode secrets** — always use `${ENV_VAR}` syntax
5. **Include a setup skill** if the MCP server requires configuration (API keys, etc.)

---

## Anti-Patterns

- **Don't** hardcode API keys or tokens in `.mcp.json`
- **Don't** use absolute paths — use `${CLAUDE_PLUGIN_ROOT}`
- **Don't** assume env vars exist — document requirements and fail gracefully
- **Don't** bundle large binaries in the plugin — use npx/uvx to install on demand
