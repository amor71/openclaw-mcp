# openclaw-mcp

Native MCP (Model Context Protocol) support for [OpenClaw](https://github.com/openclaw/openclaw).

**Status:** 📋 Requirements Phase

This is a community contribution implementing native MCP client support for OpenClaw, targeting issues [#4834](https://github.com/openclaw/openclaw/issues/4834) and [#13248](https://github.com/openclaw/openclaw/issues/13248).

## What This Adds

Users will be able to configure MCP servers in `openclaw.json` and have their tools appear alongside native OpenClaw tools — no custom skills needed.

```json
{
  "agents": {
    "defaults": {
      "mcp": {
        "servers": {
          "linkedin": {
            "command": "uvx",
            "args": ["--from", "git+https://github.com/adhikasp/mcp-linkedin", "mcp-linkedin"],
            "env": {
              "LINKEDIN_EMAIL": "you@example.com",
              "LINKEDIN_PASSWORD": "secret"
            }
          },
          "filesystem": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
          }
        }
      }
    }
  }
}
```

## Documents

| Phase | Document | Status |
|-------|----------|--------|
| Requirements | [REQUIREMENTS.md](./REQUIREMENTS.md) | 📋 In Review |
| Design | DESIGN.md | ⏳ Pending |
| Plan | PLAN.md | ⏳ Pending |
| Implementation | PR to openclaw/openclaw | ⏳ Pending |

## Process

We follow a strict development process:
1. **Requirements** — what we're building and why
2. **Design** — architecture, interfaces, data flow
3. **Plan** — implementation order, test strategy
4. **Tests** — TDD, tests before code
5. **Code** — implementation against the design

## Contributing

This is an open effort. Issues, feedback on requirements/design, and code review are all welcome.

## License

Same as OpenClaw (see [openclaw/openclaw](https://github.com/openclaw/openclaw) for license terms).
