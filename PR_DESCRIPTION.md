# feat: Native MCP (Model Context Protocol) client support

Closes #4834, #13248

## Summary

This PR adds native MCP client support to OpenClaw, allowing users to connect any MCP-compatible server and expose its tools alongside built-in tools — zero custom skills required.

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
            "env": { "GITHUB_TOKEN": "secret://env/GITHUB_TOKEN" }
          },
          "remote-api": {
            "transport": "sse",
            "url": "https://mcp.example.com/sse",
            "headers": { "Authorization": "secret://gcp/mcp-auth-token" }
          }
        }
      }
    }
  }
}
```

## What's Included

### New module: `src/mcp/`

| File | Purpose |
|------|---------|
| `config.ts` | Zod-validated config types for MCP servers |
| `secret-resolver.ts` | `secret://` URI resolution (GCP, AWS, Vault, env vars) |
| `tool-bridge.ts` | Converts MCP JSON Schema → OpenClaw AgentTool with untrusted content wrapping |
| `resource-bridge.ts` | Discovers MCP resources and injects them into agent context |
| `client-base.ts` | Abstract base class for MCP clients |
| `client-stdio.ts` | Stdio transport — child process management with crash recovery |
| `client-sse.ts` | SSE + streamable HTTP transport with reconnection |
| `manager.ts` | Lifecycle orchestrator — parallel start, aggregated tools, clean shutdown |
| `index.ts` | Public API exports |

### Integration points (surgical, minimal changes)

1. **Config schema** — Added `mcp?: McpConfig` to agent config types + Zod validation
2. **Tool assembly** (`src/agents/openclaw-tools.ts`) — MCP tools appended after plugin tools
3. **ACP translator** (`src/acp/translator.ts`) — Removed "ignoring MCP servers" stubs, wired through to McpManager
4. **External content** (`src/security/external-content.ts`) — Added `"mcp_server"` source type
5. **Gateway lifecycle** — McpManager created at startup, shutdown on stop

### Documentation

- `docs/mcp.md` — Full user-facing guide with config reference, examples, troubleshooting

## Architecture

```
Config (openclaw.json)
  → McpManager.start() [parallel per server]
    → SecretResolver (resolve secret:// URIs)
    → StdioMcpClient / SseMcpClient (connect + discover)
      → ToolBridge (MCP schema → AgentTool)
      → ResourceBridge (MCP resources → context injection)
  → Tools available as mcp_{server}_{tool}
  → All output wrapped as EXTERNAL_UNTRUSTED_CONTENT
```

See [DESIGN.md](https://github.com/amor71/openclaw-mcp/blob/main/DESIGN.md) for the full architecture document.

## Security Model

All MCP server output goes through the **same security pipeline** as `web_fetch`:

1. **Pattern detection** — Prompt injection attempts are detected and logged
2. **Marker sanitization** — Output cannot escape `EXTERNAL_UNTRUSTED_CONTENT` boundaries
3. **Untrusted wrapping** — All content clearly marked for the model
4. **Process isolation** — Stdio servers are child processes with no OpenClaw access
5. **Credential security** — `secret://` URIs resolved at runtime, never logged or written to disk

## Test Coverage

**99+ tests** across unit and integration suites:

| Test Suite | Tests | What's Covered |
|------------|-------|----------------|
| `config.test.ts` | 22 | All config permutations, validation errors, defaults |
| `tool-bridge.test.ts` | 20 | Schema conversion, naming, untrusted wrapping, error handling |
| `secret-resolver.test.ts` | 18 | All providers, parallel resolution, error cases, redaction |
| `resource-bridge.test.ts` | 12 | Discovery, filtering, context building, binary handling |
| `manager.test.ts` | 15 | Parallel start, failure isolation, aggregation, shutdown |
| `client-sse.test.ts` | 6 | SSE/HTTP connect, tool calls, auth headers |
| `client-stdio.test.ts` | 6+ | Full lifecycle, crash recovery, reconnection |
| `e2e.e2e.test.ts` | — | Full pipeline: config → start → call → wrapped result → shutdown |

## Supported Transports

| Transport | Use Case |
|-----------|----------|
| **stdio** | Local MCP servers (npm, Python, Go binaries) — default |
| **SSE** | Remote MCP servers with Server-Sent Events |
| **HTTP** | Remote MCP servers with streamable HTTP |

## Breaking Changes

**None.** This is purely additive:
- Configs without `mcp` work exactly as before
- Tool names namespaced as `mcp_*` — no collisions with existing tools
- ACP compatibility maintained (empty `mcpServers` = no-op)
- No new required dependencies (MCP SDK is the only addition)

## Dependencies

| Package | Purpose |
|---------|---------|
| `@modelcontextprotocol/sdk` ^1.x | MCP client protocol (JSON-RPC over stdio/SSE/HTTP) |

Optional (lazy-imported, not required):
- `@google-cloud/secret-manager` — for `secret://gcp/` URIs
- `@aws-sdk/client-secrets-manager` — for `secret://aws/` URIs

## How to Test

```bash
# Run MCP unit + integration tests
npx vitest run src/mcp/__tests__/

# Run full test suite (verify no regressions)
pnpm test
```

## Config Examples

<details>
<summary>Filesystem server (stdio)</summary>

```json
{
  "mcp": {
    "servers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
      }
    }
  }
}
```
</details>

<details>
<summary>GitHub server (Python/uvx)</summary>

```json
{
  "mcp": {
    "servers": {
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
```
</details>

<details>
<summary>Remote SSE server with auth</summary>

```json
{
  "mcp": {
    "servers": {
      "remote": {
        "transport": "sse",
        "url": "https://mcp.example.com/sse",
        "headers": {
          "Authorization": "secret://gcp/mcp-auth-token"
        }
      }
    }
  }
}
```
</details>

<details>
<summary>Multiple servers + per-agent config</summary>

```json
{
  "agents": {
    "defaults": {
      "mcp": {
        "servers": {
          "filesystem": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "/shared"]
          }
        }
      }
    },
    "list": [
      {
        "name": "dev-agent",
        "mcp": {
          "servers": {
            "github": {
              "command": "uvx",
              "args": ["mcp-server-github"],
              "env": { "GITHUB_TOKEN": "secret://env/GITHUB_TOKEN" }
            }
          }
        }
      }
    ]
  }
}
```
</details>
