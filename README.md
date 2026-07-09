# Aient MCP

**MCP-native AI SRE: ask what's broken in production, get a reviewed GitHub fix PR.**

[Aient AI](https://aient.ai) ingests your OpenTelemetry logs, metrics, and traces, groups errors into problems, investigates root cause, and opens a GitHub pull request with a fix — then watches production telemetry to check the problem stops recurring. The pull request is the human approval gate; Aient does not merge or deploy to your production itself.

This repository documents Aient's **Model Context Protocol (MCP) server** and how to connect it from Claude, Codex, Cursor, VS Code / Copilot, and other MCP clients. The server itself is hosted — there is nothing to install or self-host.

- Homepage: https://aient.ai
- Integration docs: https://docs.aient.ai
- MCP server URL: `https://aient.ai/mcp`
- Listed in the [official MCP Registry](https://registry.modelcontextprotocol.io/v0/servers?search=aient) as `ai.aient/mcp` and on [Glama](https://glama.ai/mcp/servers/ai.aient)

> Status: commercial, early-access, closed-source. Telemetry you send is ingested and processed by Aient.

## What the server exposes

Aient MCP lets an MCP-capable client (IDE agent, chat agent, autonomous agent) query your Aient telemetry and, with write consent, drive operational actions such as remediation and scoped key management.

A representative set of tools (start with `verify_connection`):

| Tool | Purpose |
| --- | --- |
| `verify_connection` | Confirm the token, scoped organisation, granted scopes, and live tool count. |
| `describe_telemetry` | Discover services, operations, and attributes before narrower queries. |
| `service_health_summary` | Per-service error rate, p99 latency, active problems and remediations. |
| `query_traces` / `query_logs` | Aggregate traces/logs by service, status, severity, or time bucket. |
| `get_spans` / `get_logs` | Fetch raw spans / log entries for detailed inspection. |
| `detect_anomalies` | Flag anomalous error-rate / latency behaviour. |
| `list_problems` / `get_problem` | Read Aient-detected problems with stack traces and sample trace IDs. |
| `request_problem_remediation` | Start an AI remediation run that opens a fix PR. |

The MCP server is the source of truth for exact tool schemas — see the [tool reference](https://docs.aient.ai) for the full list.

## Transports

Aient exposes two MCP transports with the same tool families and different principals:

| Transport | URL | Principal | Use when |
| --- | --- | --- | --- |
| OAuth | `https://aient.ai/mcp` | Browser OAuth (human-owned Aient org) | A human-owned organisation is set up or can complete browser OAuth. |
| x402 (paid) | `https://aient.ai/mcp-x402` | Wallet address, x402 USDC per priced call | An autonomous agent has a wallet and wants agent-native access without OAuth. |

Both are **remote streamable-HTTP** (SSE is disabled). The transports do not fall through to each other. Most humans and coding agents want the OAuth transport below; for the wallet flow see the [agentic x402 onboarding docs](https://docs.aient.ai/agentic-x402).

## Authentication

The OAuth transport uses OAuth 2.0 (Authorization Code) with Protected Resource Metadata discovery:

- Protected resource metadata: `https://aient.ai/.well-known/oauth-protected-resource`
- Authorization server metadata: `https://aient.ai/.well-known/oauth-authorization-server`
- Scopes: `aient.mcp.read` (read floor), `aient.mcp.onboarding` (narrow setup), `aient.mcp.write` (mutating actions)

Your client must support **adding a remote MCP server by URL** and a **browser OAuth flow**. Clients that can't do OAuth for remote MCP servers will see `401`. For those, use the [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) bridge shown below.

## Connect from your client

### Claude Code

```bash
claude mcp add --transport http aient https://aient.ai/mcp
```

Start a session and ask Claude to run an Aient tool (e.g. *"Run `service_health_summary` for the last 30 minutes"*). Claude Code opens a browser for sign-in and consent.

### Claude Desktop

Add a custom connector in **Settings → Connectors** with URL `https://aient.ai/mcp`, or use the `mcp-remote` bridge in `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "aient": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://aient.ai/mcp"]
    }
  }
}
```

### Cursor

`.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "aient": {
      "url": "https://aient.ai/mcp"
    }
  }
}
```

Cursor connects and, on supported versions, opens a browser for OAuth sign-in and consent.

### VS Code (GitHub Copilot)

`.vscode/mcp.json`:

```json
{
  "servers": {
    "aient": {
      "type": "http",
      "url": "https://aient.ai/mcp"
    }
  }
}
```

On a VS Code / Copilot build with OAuth support for remote MCP servers, it opens a browser for sign-in and then exposes the Aient tools to Copilot.

### Codex

Codex is stdio-based, so bridge the remote server with `mcp-remote` in `~/.codex/config.toml`:

```toml
[mcp_servers.aient]
command = "npx"
args = ["-y", "mcp-remote", "https://aient.ai/mcp"]
```

`mcp-remote` completes the OAuth flow and keeps the token refreshed. (Codex refresh-token recovery has historically been unreliable — if you hit `OAuth authorization required` on startup, re-run login.)

### Any other MCP client

Point it at `https://aient.ai/mcp` if it supports remote OAuth MCP, otherwise wrap it with the universal bridge:

```bash
npx -y mcp-remote https://aient.ai/mcp
```

## Verifying it works

After connecting, run `verify_connection`. It returns the scoped organisation, granted scopes, token expiry, and the live tool count. If `activation.completed` is `false`, run `get_activation_status` and follow its `nextActions` to finish setup.

## Links

- Product: https://aient.ai
- Docs: https://docs.aient.ai
- MCP feature reference: https://docs.aient.ai (search "MCP")
- Agentic x402 onboarding: https://docs.aient.ai/agentic-x402
- MCP Registry entry: `ai.aient/mcp`
- Glama: https://glama.ai/mcp/servers/ai.aient

## License

MIT © Triple Alpha AB. See [LICENSE](LICENSE). The Aient service itself is proprietary; this repository contains only documentation and connection configuration.
