# Moss Skills

Skills and rules for using [Moss](https://github.com/hpungsan/moss) with AI coding tools.

Moss is a local context capsule store for portable AI handoffs, multi-agent orchestration, and cross-tool context sharing. These files teach AI tools how to use Moss correctly — capsule format, addressing modes, orchestration patterns, and error handling.

## Supported Tools

| Tool | Path | Status |
|------|------|--------|
| Claude Code | `.claude/skills/moss/` | Available |
| Codex | — | Planned |

## Setup

### Claude Code

Copy `.claude/skills/moss/` into your project's `.claude/skills/` directory.

```bash
cp -r .claude/skills/moss/ /path/to/your/project/.claude/skills/moss/
```

Requires [Moss](https://github.com/hpungsan/moss) MCP server configured in your project's `.mcp.json`.

## Links

- [Moss](https://github.com/hpungsan/moss) — Core tool

