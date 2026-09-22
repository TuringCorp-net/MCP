# TuringCorp Decider — MCP server

A **judge for agent decisions**. Your agent has two defensible options and has to pick one. Send both to Decider; it comes back with which one it prefers, **how far apart it judged them** (a calibrated confidence), and why.

- **Endpoint (canonical):** `https://mcp.turingcorp.net/mcp`
- **Aliases:** `https://mcp.turingcorp.net/` and `https://mcp.turingcorp.net/mcp/`
- **Health:** `https://mcp.turingcorp.net/healthz`
- **Transport:** Streamable HTTP, **stateless** (no session, no Durable Object). `GET /mcp` → 405 is normal.
- **Protocol:** MCP `2026-07-28` (modern) **and** `2025-06-18` (legacy) — one route serves both.
- **Discovery methods** (`server/discover`, `tools/list`) need **no credentials** — directory scanners can read the full tool list unauthenticated, by design.

## The one tool: `decide`

| | |
|---|---|
| **In** | `task` (string), `option_a` (string), `option_b` (string) — all required |
| **Out** | `better_option` (`"A"` \| `"B"`), `confidence` (percentage string, e.g. `"83.3%"`), `reason` (string) |
| **Annotations** | `readOnlyHint: true` · `openWorldHint: false` · `idempotentHint: false` |

`confidence` is **this service's own judgement of how far apart the two options were** — a reference for your decision-making, not an instruction, not a result, and not a prediction of how the choice turns out. As a rough reading: ≥90% clearly apart · 80–89% apart, less clearly · 70–79% a closer call · <70% close to evenly matched. These ranges are descriptive only. **Choose your own threshold for your own use case; for high-stakes or irreversible decisions apply your own review policy.** Observed accuracy by range, and how it was measured, is published at <https://api.turingcorp.net>.

⚠️ `idempotentHint: false` is an honest declaration: there is currently no idempotency key, so a client that times out and retries **may be charged twice**.

## Authentication

Send an **Agent Pass**: `Authorization: Bearer <pass>`.

Issued at **<https://agent-pass.turingcorp.net>** — self-service signup (email verification) → top up → get a pass. A pass is valid for **7 days** and can be re-rolled. If a call is refused as an invalid credential, sign in there again and re-roll.

| Situation | Response |
|---|---|
| No credential | `401` + `WWW-Authenticate: Bearer realm="turingcorp-mcp"` |
| Expired / invalid pass | `invalid_credential`, with a re-login URL in the message |
| Insufficient balance | `402` + `action_url` pointing at top-up |
| Over quota | `429` + `Retry-After` |

Entry rate limit: **120 requests / 60 s / client IP**. This is flood damping, not a quota — the real per-key quota is enforced separately. The counter is approximate: it is maintained per edge location and is eventually consistent, so a brief overshoot is possible.

## Quickstart

You need an **Agent Pass** to call the tool. Discovery (`tools/list`) works without one — calling `decide` does not.

### 1. Get an Agent Pass

Self-service at **<https://agent-pass.turingcorp.net>**: sign up with an email address, verify it, add credit. The pass is shown once; it is valid for **7 days** and can be re-rolled at any time. Agents can do the same thing over the API — see that site's `/llms.txt`.

### 2. Point your client at the endpoint

**Claude Desktop / Cursor / any client that reads a JSON config:**

```json
{
  "mcpServers": {
    "decider": {
      "command": "npx",
      "args": ["-y", "mcp-remote@latest", "https://mcp.turingcorp.net/mcp"],
      "env": { "AUTHORIZATION": "Bearer <your Agent Pass>" }
    }
  }
}
```

> ⚠️ A bare `npx -y mcp-remote@latest https://mcp.turingcorp.net/mcp` with no credential **will connect and list tools, then fail on the first call** with 401. That is expected — discovery is open, execution is not.

**Claude Code:**

```bash
claude mcp add --transport http decider https://mcp.turingcorp.net/mcp \
  --header "Authorization: Bearer <your Agent Pass>"
```

### 3. Sanity-check it yourself

**Discovery — no credentials needed:**

```bash
curl -s -X POST https://mcp.turingcorp.net/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -H 'mcp-protocol-version: 2026-07-28' \
  -H 'mcp-method: tools/list' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{}}}}'
```

**Calling the tool** additionally needs `mcp-name: decide` and the credential:

```bash
curl -s -X POST https://mcp.turingcorp.net/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -H 'mcp-protocol-version: 2026-07-28' \
  -H 'mcp-method: tools/call' \
  -H 'mcp-name: decide' \
  -H "authorization: Bearer $AGENT_PASS" \
  --data '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"decide","arguments":{"task":"Pick a launch date","option_a":"Ship now","option_b":"Wait two weeks"},"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{}}}}'
```

### Protocol details worth knowing

- **`Accept` must contain both** `application/json` **and** `text/event-stream`, or you get 406.
- **Protocol versions:** one route serves both generations.
  - **Modern `2026-07-28`** — no `initialize` handshake; use `server/discover` and `_meta`.
  - **Legacy** — standard `initialize` handshake. Measured negotiation: a client asking for `2025-11-25`, `2025-06-18`, or `2025-03-26` is answered with **exactly the version it asked for**; a modern `2026-07-28` `initialize` (which is not part of that generation) is answered with `2025-11-25`.
  - Server capability discovery via `tools/list` and `server/discover` needs no credentials.
- **Errors are machine-readable.** When a call fails, the tool result carries a prose block **and** a second block containing a single JSON object, e.g. `{"error":"invalid_credential","http_status":"401","action_url":"…"}`. **Branch on `error`, not on the message.**
- **Authorization discovery:** `401` responses carry a `resource_metadata` pointing at `/.well-known/oauth-protected-resource`. Note that this server uses a **static bearer credential, not OAuth** — that document says so explicitly rather than sending you into an OAuth flow that does not exist.

## Not for browser clients

This endpoint deliberately does **not** serve browser-based (cross-origin) MCP clients: requests carrying a browser `Origin` header other than localhost are rejected with 403, and no CORS headers are returned. Browser tooling such as the MCP Inspector works by proxying through a **local** process, which sends no `Origin` — that path works normally. If you need a browser page to reach this server, run a proxy you control rather than calling it cross-origin.

## What this is not

- **One tool only.** `decide` is the entire surface. There is no way to reach other tiers through this endpoint.
- **No SLA.** No availability commitment is offered, and none should be inferred.
- **Not an autopilot.** Decider is a component you call. How you gate on it — thresholds, human review, retries — is your policy and stays yours.
- **No idempotency** (see above).

## About this repository

This repository carries the **discovery metadata** for the server above — what directory services and clients read to find and describe it.

- [`server.json`](server.json) — the official MCP Registry manifest
- [`docs/DIRECTORIES.md`](docs/DIRECTORIES.md) — where this server is listed, and what each directory requires

The server itself is a hosted remote endpoint; nothing here needs to be installed or run.

## License

See [LICENSE](LICENSE).
