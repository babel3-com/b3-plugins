# Session Memory Configuration Guide

## MCP Server Setup

The session-memory MCP server runs as `b3 mcp session-memory` (stdio transport). It ships as part of the b3 open-source plugin.

### Plugin MCP Entry

In `.mcp.json`:

```json
{
  "session-memory": {
    "command": "b3",
    "args": ["mcp", "session-memory"],
    "env": {
      "SESSION_EXTRA_DIRS": "",
      "SESSION_INDEX_FILE": "",
      "SESSION_AGENT_NAME": ""
    }
  }
}
```

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `SESSION_DIRS` | *(auto-discover)* | Colon-separated list of directories containing session JSONL files. Overrides auto-discovery. |
| `SESSION_EXTRA_DIRS` | `""` | Additional colon-separated dirs to search. Appended to auto-discovered dirs. |
| `SESSION_INDEX_FILE` | `""` | Path to session index JSON file (for tag enrichment). Optional. |
| `SESSION_AGENT_NAME` | *(from $USER)* | Override agent display name. Default: capitalize after `agent-` prefix. |

## Auto-Discovery

When `SESSION_DIRS` is not set, the server scans `~/.claude/projects/` for subdirectories containing `.jsonl` files.

## Remote Sessions

For remote sessions (e.g. from `.claude-remote/projects/`), add the path to `SESSION_EXTRA_DIRS`:

```json
{
  "session-memory": {
    "command": "b3",
    "args": ["mcp", "session-memory"],
    "env": {
      "SESSION_EXTRA_DIRS": "/path/to/project/.claude-remote/projects/-project-path"
    }
  }
}
```

## Multi-Agent Setup

Each agent instance runs its own session-memory MCP. `SESSION_AGENT_NAME` controls the display name in resurfaced output.

To cross-reference another agent's sessions, add their session directory to `SESSION_EXTRA_DIRS`:

```json
{
  "session-memory": {
    "command": "b3",
    "args": ["mcp", "session-memory"],
    "env": {
      "SESSION_EXTRA_DIRS": "/home/agent-zara/.claude/projects/-home-agent-zara-dev-heycode"
    }
  }
}
```
