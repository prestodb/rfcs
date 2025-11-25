# **RFC-0022 for Presto**

## Presto Model Context Protocol Support

Proposers

* Reetika Agrawal

## Summary

This proposal introduces support for integrating Model Context Protocol (MCP) with Presto through a separate lightweight process, referred to as the MCP Server.
The MCP Server acts as a protocol translation layer between AI agents using JSON-RPC and Presto’s existing HTTP-based query protocol.

## Background

Presto exposes a RESTful HTTP API for query submission and result retrieval. Clients submit a query via a POST request and receive a tokenized nextUri chain for incremental result fetching.
However, AI frameworks like OpenAI MCP operate on a request–response model over JSON-RPC, where clients expect complete results in a single response, not streamed batches.
Because MCP and Presto differ fundamentally in communication models, a direct integration is impractical. Additionally, direct embedding of MCP into Presto coordinators would disrupt routing and proxy layers that rely on Presto’s native HTTP semantics.

### Goals

- Enable MCP-compatible AI agents (e.g., OpenAI ChatGPT tools) to query Presto seamlessly.
- Preserve all existing Presto router, proxy, and load-balancing infrastructure without modification.
- Simplify authentication by reusing existing OAuth/JWT mechanisms.
- Prevent unbounded queries by applying automatic limits when appropriate.

### Proposed Plan

Introduce a new lightweight service: presto-mcp-server, deployed alongside Presto coordinators and Presto Router.

The MCP server will:

- Expose a JSON-RPC 2.0 HTTP endpoint (`/mcp`, `/v1/mcp`) interface to AI clients.
- Implement the core MCP primitives:
  - `tools/list` for tool discovery 
  - `tools/call` for executing tools
  - Initially provide a single tool: `query.run`, which executes SQL queries against Presto.
- Internally communicate with Presto coordinators using standard Presto HTTP APIs.
- Forward OAuth/JWT Bearer tokens transparently from MCP clients to Presto, ensuring that Presto performs all authentication and authorization checks.
- Translate between the two protocols, aggregating streaming results into a single response.
- Remain stateless, delegating all query lifecycle management to Presto.

## Proposed Implementation

#### Core Changes

```json
JsonRpcServlet  → McpDispatcher → ToolRegistry → QueryRunTool → PrestoQueryClient → Presto Coordinator
```

1. New Module: `presto-mcp-server`

   - Implements JSON-RPC 2.0 protocol.
   - Implements core MCP primitive like `tools/list` and `tools/call`
   - Handles methods like query.run.

2. On `query.run`:

   - Parses SQL input.
   - Optionally injects a LIMIT clause (if absent) to control data size.
   - Submits the query to Presto coordinator via /v1/statement.
   - Polls the returned nextUri until the query completes.
   - Returns final aggregated results as a single JSON-RPC response.

#### Example Queries

- `tools/list`
```json
{
"jsonrpc": "2.0",
"id": 1,
"method": "tools/list",
"params": {}
}
```

- `tools/call → query.run`
```json
{
"jsonrpc": "2.0",
"id": 2,
"method": "tools/call",
"params": {
  "name": "query.run",
  "arguments": { "sql": "SELECT 1" }
  }
}
```

### Rationale

  - Introduce a new standalone service (presto-mcp-server) to avoid mixing JSON-RPC with Presto’s stateful HTTP protocol.
  - Translate MCP tool calls into Presto HTTP queries, using the existing StatementClient to follow nextUri pages and aggregate results into a single MCP response.
  - Keep the MCP server stateless, with all query lifecycle state remaining on Presto coordinators.
  - Forward OAuth/JWT Bearer tokens directly from MCP clients to Presto, allowing Presto to perform full authentication and authorization without changes.
  - Preserve all existing Presto infrastructure (Router, proxies) by keeping MCP outside the coordinator and communicating using standard Presto HTTP APIs.

## Backward Compatibility Considerations

  - MCP server is a new optional component
  - No impact on existing Presto or Router
  - All existing Presto deployments remain unchanged

## Test Plan

- **Unit + Integration Tests:**
Verify ToolRegistry loading, dispatcher routing, SQL execution via QueryRunTool, and JSON-RPC error handling.

- **End-to-End Validation:**
Deploy MCP server with Router + Coordinator. Confirm Bearer token forwarding works and MCP clients (Gemini, ChatGPT) successfully execute queries.

## Modules involved
- `presto-mcp` (new module)
- `presto-client`
- `airlift` modules
- `presto-main`
- `presto-router`

## Final Thoughts

This proposal cleanly bridges Presto with next-generation agent ecosystems (LLMs, AI workflows, model interaction tools). The MCP server architecture respects Presto’s deployment patterns, is backward-compatible, and provides a robust extension point for future interactive functionality such as:

- Schema browsing tools
- Table metadata tools
- Query explanation tools

## WIP - Draft PR Changes
