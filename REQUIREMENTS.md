# OpenClaw Native MCP Support — Requirements Document

**Author:** Rye (amor71/openclaw fork)
**Date:** February 19, 2026
**Target Issues:** #4834, #13248
**Status:** DRAFT — Awaiting Review

---

## 1. Problem Statement

OpenClaw currently has no user-facing MCP (Model Context Protocol) support. The ACP translator explicitly ignores MCP server configurations passed by clients. Users cannot connect to external MCP servers (LinkedIn, Notion, filesystem, databases, etc.) through OpenClaw's config.

The MCP ecosystem has grown rapidly since Anthropic's November 2024 release. Hundreds of MCP servers exist for popular services. OpenClaw users are locked out of this ecosystem — they must build custom skills or use browser automation for each integration.

### 1.1 Current State

| Component | Status | Details |
|-----------|--------|---------|
| `mcporter` | Internal only | Used for doc search, not user-facing |
| `mcpServers` in config | Ignored | ACP translator logs "ignoring X MCP servers" |
| `mcpCapabilities.http` | Disabled | Returns `false` |
| `mcpCapabilities.sse` | Disabled | Returns `false` |
| `agents.defaults.mcp` | Invalid | Not in config schema |

### 1.2 Community Demand

- **Issue #4834** — 2023, "Feature: Native MCP support" — high priority, active discussion
- **Issue #13248** — 2026, "Full MCP Support" — detailed analysis of gaps, proposed config
- **PR #7148** — "feat(acp): enable MCP tool capabilities" — closed, not merged
- **PR #9829** — "Fix MCP transport reconnect and SSE" — open
- **PR #9605** — "feat(qmd): use MCP server mode" — open

---

## 2. Goals

### 2.1 Primary Goals

1. **Users can configure MCP servers in `openclaw.json`** — per-agent or in defaults
2. **MCP server tools appear as agent tools** — the LLM sees and calls them like any native tool
3. **Support stdio transport** — the most common MCP transport (command + args + env)
4. **MCP servers start/stop with agent sessions** — proper lifecycle management
5. **Tool calls route correctly** — agent tool calls → MCP server → results back to agent

### 2.2 Secondary Goals (Nice to Have)

6. **SSE/HTTP transport** — connect to remote MCP servers via URL
7. **Resource support** — expose MCP resources as agent context
8. **Prompt support** — expose MCP prompt templates
9. **Hot reload** — restart MCP servers on config change without restarting OpenClaw

### 2.3 Non-Goals (Out of Scope for v1)

- MCP **server** mode (OpenClaw exposing its tools via MCP to other clients)
- MCP sampling (letting MCP servers request LLM completions)
- MCP roots/notifications beyond basic lifecycle
- Custom MCP transport implementations
- GUI/wizard for MCP server discovery

---

## 3. User Stories

### US-1: Configure an MCP Server
**As** an OpenClaw user,
**I want to** add an MCP server to my `openclaw.json`,
**So that** my agent can use its tools without writing custom skills.

**Acceptance Criteria:**
- Config accepts `mcp.servers` with name, command, args, env
- Invalid config (missing command, invalid transport) produces clear error on startup
- Config validation follows existing OpenClaw schema patterns

### US-2: Use MCP Tools in Conversation
**As** an OpenClaw user,
**I want** MCP server tools to appear alongside native tools,
**So that** the agent can call them naturally during conversation.

**Acceptance Criteria:**
- MCP tools show up in the agent's tool list with name, description, parameters
- Agent can call MCP tools and receive results
- Tool names are prefixed/namespaced to avoid collisions with native tools (e.g., `mcp_linkedin_get_feed`)
- Tool errors from MCP servers are handled gracefully (not crash the agent)

### US-3: MCP Server Lifecycle
**As** an OpenClaw operator,
**I want** MCP servers to start automatically and restart on failure,
**So that** I don't have to manage them manually.

**Acceptance Criteria:**
- MCP servers spawn on gateway start (or first session creation)
- Crashed MCP servers are restarted with backoff
- Clean shutdown on gateway stop
- Server status visible in logs/diagnostics

### US-4: Per-Agent MCP Configuration
**As** an OpenClaw user with multiple agents,
**I want** different agents to have different MCP servers,
**So that** each agent has access to only what it needs.

**Acceptance Criteria:**
- MCP servers can be configured in `agents.defaults.mcp` (all agents) or per-agent
- Per-agent config overrides defaults
- Agent tool policy (`toolPolicy`) applies to MCP tools too

### US-5: Security & Isolation
**As** an OpenClaw operator,
**I want** MCP servers to be isolated and auditable,
**So that** a malicious MCP server can't compromise the system.

**Acceptance Criteria:**
- MCP server processes run with the same permissions as OpenClaw (no escalation)
- Tool calls to MCP servers are logged
- Env vars in config support secrets references (existing OpenClaw secret patterns)
- MCP tool execution respects existing tool policy (allow/deny lists)

---

## 4. Functional Requirements

### FR-1: Configuration Schema

```json
{
  "agents": {
    "defaults": {
      "mcp": {
        "servers": {
          "linkedin": {
            "enabled": true,
            "command": "uvx",
            "args": ["--from", "git+https://github.com/adhikasp/mcp-linkedin", "mcp-linkedin"],
            "env": {
              "LINKEDIN_EMAIL": "user@example.com",
              "LINKEDIN_PASSWORD": "secret"
            },
            "transport": "stdio"
          },
          "filesystem": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"],
            "transport": "stdio"
          }
        }
      }
    },
    "list": [
      {
        "id": "research",
        "mcp": {
          "servers": {
            "tavily": {
              "url": "https://mcp.tavily.com/mcp",
              "transport": "sse",
              "headers": {
                "Authorization": "Bearer ${TAVILY_API_KEY}"
              }
            }
          }
        }
      }
    ]
  }
}
```

**Schema Details:**
- `servers`: Object keyed by server name (used for namespacing)
- `enabled`: Optional boolean, default `true`
- `command`: Required for stdio transport — executable path
- `args`: Optional string array — command arguments
- `env`: Optional object — environment variables for the process
- `transport`: `"stdio"` (default) | `"sse"` | `"http"`
- `url`: Required for sse/http transport — server URL
- `headers`: Optional object — HTTP headers for sse/http transport
- `timeout`: Optional number — connection timeout in ms (default 30000)
- `restartOnCrash`: Optional boolean — auto-restart on crash (default `true`)
- `maxRestarts`: Optional number — max restarts before giving up (default 5)
- `toolPrefix`: Optional string — override auto-prefix for tool names

### FR-2: Tool Bridging

- Each MCP server's tools are discovered via `tools/list` on connection
- Tool names are prefixed: `mcp_{serverName}_{toolName}` (configurable)
- MCP tool JSON Schema parameters → TypeBox/OpenClaw parameter schema
- Tool descriptions include the server name for clarity
- Tool results (text/image content) → OpenClaw `AgentToolResult` format

### FR-3: Lifecycle Management

- **Startup:** Spawn configured MCP servers when gateway starts
- **Health:** Monitor server process; detect crashes via exit/error events
- **Restart:** Exponential backoff: 1s, 2s, 4s, 8s, 16s, max 30s
- **Shutdown:** Send SIGTERM, wait 5s, SIGKILL if needed
- **Lazy start (optional):** Only start server when its tools are first called

### FR-4: Error Handling

- MCP server crash → log error, attempt restart, mark tools as unavailable during restart
- Tool call to unavailable server → return error message to agent (not throw)
- Timeout on tool call → configurable, default 60s
- Invalid tool response → return error with details to agent

### FR-5: Logging & Diagnostics

- MCP server start/stop/restart events logged at `info` level
- Tool calls logged at `debug` level (with timing)
- Server stderr captured and logged at `warn` level
- Diagnostic info available via `openclaw status` or equivalent

---

## 5. Non-Functional Requirements

### NFR-1: Performance
- MCP server startup should not block gateway startup (parallel init)
- Tool call overhead (bridge latency) < 50ms on top of actual server response time
- Idle MCP servers should consume minimal resources

### NFR-2: Compatibility
- Must work with any MCP server that implements the 2024-11-05 spec
- Must work on Linux, macOS, Windows (same as OpenClaw)
- Must not break existing tool functionality

### NFR-3: Security
- No privilege escalation through MCP servers
- Environment variable secrets never logged
- MCP servers inherit OpenClaw's process security context
- Tool policy enforcement applies uniformly to MCP tools

### NFR-4: Testability
- Unit tests for config validation, tool bridging, lifecycle
- Integration tests with a mock MCP server (stdio)
- E2E test with a real simple MCP server (e.g., filesystem)

---

## 6. Dependencies

| Dependency | Purpose | Status |
|-----------|---------|--------|
| `@modelcontextprotocol/sdk` | MCP client implementation | npm package, stable |
| OpenClaw config schema | Add `mcp` section | Needs modification |
| `@mariozechner/pi-agent-core` | `AgentTool` interface | Existing, no changes needed |
| TypeBox | Parameter schema conversion | Existing in OpenClaw |

---

## 7. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| MCP spec evolves, breaking changes | Medium | Pin to 2024-11-05 spec version, add version negotiation |
| Malicious MCP server | High | Sandboxing, tool policy, audit logging |
| MCP server memory leaks | Medium | Process isolation (separate processes), restart policy |
| Tool name collisions | Low | Mandatory namespacing with configurable prefix |
| OpenClaw maintainers reject PR | Medium | Align with existing patterns, reference open issues, clean code |
| ACP protocol changes | Low | MCP bridge is independent of ACP layer |

---

## 8. Open Questions

1. **Tool namespacing strategy** — Should it be `mcp_server_tool` (verbose but clear) or `server_tool` (shorter)? Or let users configure?
2. **Resource/prompt support** — Should v1 support MCP resources (injecting into context) or defer entirely?
3. **Config location** — `agents.defaults.mcp` vs top-level `mcp` section? Per-agent is cleaner but top-level is simpler.
4. **Lazy vs eager startup** — Start all servers on boot, or only when needed? Lazy saves resources but adds latency on first call.
5. **Interaction with tool policy** — How do MCP tools interact with existing `toolPolicy` allow/deny lists? By prefixed name?

---

## 9. Success Criteria

- [ ] User can add an MCP server to `openclaw.json` and use its tools in conversation
- [ ] At least stdio transport works end-to-end
- [ ] MCP server crashes don't crash OpenClaw
- [ ] Tool policy applies to MCP tools
- [ ] All tests pass, including existing test suite (no regressions)
- [ ] PR references #4834 and #13248
- [ ] Documentation updated (config reference, example configs)

---

## 10. Timeline Estimate

| Phase | Duration |
|-------|----------|
| Requirements (this doc) | 1 day |
| Design document | 1-2 days |
| Test scaffolding | 1 day |
| Implementation | 3-5 days |
| Integration testing | 1-2 days |
| Documentation + PR | 1 day |
| **Total** | **~8-12 days** |
