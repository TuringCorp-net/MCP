# Where this server is listed

The MCP directory landscape changes quickly, and each directory has its own submission
requirements. This page records **where this server is listed and what each directory expects**,
so integrations can rely on a single canonical endpoint.

## Canonical identity

| | |
|---|---|
| Registry name | `net.turingcorp.mcp/decider` |
| Endpoint | `https://mcp.turingcorp.net/mcp` |
| Landing page | `https://mcp.turingcorp.net/` |
| Registry manifest | [`server.json`](../server.json) |
| Auth | static bearer credential (Agent Pass) |

## Listings

| Directory | Status | Requirement (as verified 2026-09-22) |
|---|---|---|
| **Official MCP Registry** | ✅ Listed — `active` | Publish `server.json` with the `mcp-publisher` CLI. Domain ownership is proven either by a DNS `TXT` record **or** by an HTTP document at `/.well-known/mcp-registry-auth`; this server uses the **HTTP** method, so no DNS change is required. Remote servers declare auth natively via `remotes[].headers[]` (`isRequired` / `isSecret`) — **OAuth is not required**. The registry is still in preview and may change or reset. |
| **Smithery** | ⏳ Pending account | Publish by entering the server's public HTTPS URL at `smithery.ai/new`; Smithery proxies to the upstream. Requires Streamable HTTP, and OAuth support *if* auth is required — this server uses a static bearer credential instead. Scans requests with user agent `SmitheryBot/1.0`; a static server card at `/.well-known/mcp/server-card.json` is the documented fallback when scanning cannot complete. |
| **mcp.so** | ⏳ To do | Submit a one-line README PR to the `chatmcp/mcpso` repository. |
| **awesome-mcp-servers and similar lists** | ⏳ To do | Requires the project repository to be public. |
| **Glama** | — | Crawls public GitHub repositories and additionally expects a **local stdio entry point** inside the repository; a URL-only server does not qualify. |

## What a scanner sees

Verified against this deployment's live endpoint on 2026-09-22:

| Probe | Result |
|---|---|
| unauthenticated `tools/list` | `200` — the full tool list is readable without credentials, by design |
| unauthenticated `initialize` (legacy `2025-06-18`) | `200` |
| unauthenticated `tools/call` | `401` + `WWW-Authenticate` (directories expect 401 rather than 403) |
| user agent `SmitheryBot/1.0` | not blocked |

Discovery being open is deliberate: directory scanners must be able to enumerate capabilities
before anyone has a credential. Execution is not open.

## Notes for directory maintainers

- One tool only: `decide`. It is described in full in the [README](../README.md).
- The server card and this manifest are generated from the live server, so they cannot drift
  from what the endpoint actually serves.
- If a scan fails against your infrastructure, the failure is almost always bot protection on
  the caller's side; the endpoint itself answers unauthenticated discovery normally.
