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
| **Smithery** | ✅ Listed — `turingcorp/mcp` | Published by URL; Smithery proxies to the upstream at `https://mcp--turingcorp.run.tools`. Requires Streamable HTTP. Scans requests carry user agent `SmitheryBot/1.0`. Two things worth knowing: (1) Smithery reads this server's **static server card** (`/.well-known/mcp/server-card.json`) for the tool inventory, so the card and the live `tools/list` must stay identical; (2) Smithery's gateway authenticates clients with **its own** token and forwards a user's credential upstream only when the listing declares a configuration schema — this server uses a static bearer credential, so an Agent Pass has to be supplied by the client. |
| **mcp.so** | — Declined | Listing is paid-only and the free path was withdrawn; not pursued. |
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
