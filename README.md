# openclaw-mcp

Native MCP (Model Context Protocol) support for [OpenClaw](https://github.com/openclaw/openclaw).

**Status:** ✅ Implementation Complete — PR Ready

This is a community contribution implementing native MCP client support for OpenClaw, targeting issues [#4834](https://github.com/openclaw/openclaw/issues/4834) and [#13248](https://github.com/openclaw/openclaw/issues/13248).

## What This Adds

Users can configure MCP servers in `openclaw.json` and have their tools appear alongside native OpenClaw tools — no custom skills needed.

```json
{
  "agents": {
    "defaults": {
      "mcp": {
        "servers": {
          "filesystem": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
          },
          "github": {
            "command": "uvx",
            "args": ["mcp-server-github"],
            "env": {
              "GITHUB_TOKEN": "secret://env/GITHUB_TOKEN"
            }
          }
        }
      }
    }
  }
}
```

## Architecture

See [DESIGN.md](./DESIGN.md) for the full architecture. Key components:

| Module | Purpose |
|--------|---------|
| `config.ts` | Zod-validated config schema for MCP servers |
| `secret-resolver.ts` | `secret://` URI resolution (GCP, AWS, Vault, env) |
| `tool-bridge.ts` | MCP tool schema → OpenClaw AgentTool conversion |
| `resource-bridge.ts` | MCP resource discovery + context injection |
| `client-stdio.ts` | Stdio transport with crash recovery |
| `client-sse.ts` | SSE/HTTP transport with reconnection |
| `manager.ts` | Lifecycle orchestrator for all MCP servers |

## Implementation Status

| Phase | Description | Status |
|-------|-------------|--------|
| 0 | Scaffolding & CI | ✅ Done |
| 1 | Config & Validation | ✅ Done (22 tests) |
| 2 | Client Lifecycle | ✅ Done |
| 3 | Tool Bridge | ✅ Done (20 tests) |
| 4 | Resource Bridge | ✅ Done (12 tests) |
| 5 | Secret Resolver | ✅ Done (18 tests) |
| 6 | Manager | ✅ Done (15 tests) |
| 7 | Integration Wiring | ✅ Done |
| 8 | E2E & Security Tests | ✅ Done |
| 9 | Documentation | ✅ Done |
| 10 | PR Preparation | ✅ Done |

**Test coverage:** 99+ unit/integration tests passing across all MCP modules.

## Documents

| Document | Description |
|----------|-------------|
| [REQUIREMENTS.md](./REQUIREMENTS.md) | Functional and non-functional requirements |
| [DESIGN.md](./DESIGN.md) | Architecture, component design, security model |
| [PLAN.md](./PLAN.md) | Phased implementation plan |
| [PR_DESCRIPTION.md](./PR_DESCRIPTION.md) | Pull request description |

## Key Design Decisions

- **Security first:** All MCP output wrapped as untrusted external content (same model as `web_fetch`)
- **Secret management:** `secret://` URIs for credentials — never stored in plaintext logs
- **Resilient:** Auto-restart with exponential backoff for crashed servers
- **Non-breaking:** Purely additive — no changes to existing behavior without `mcp` config
- **Namespaced tools:** `mcp_{server}_{tool}` naming prevents collisions

## License

Same as OpenClaw (see [openclaw/openclaw](https://github.com/openclaw/openclaw) for license terms).
