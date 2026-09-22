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

**Claude Code / Cursor / any `mcp-remote`-based client:**

```bash
npx -y mcp-remote@latest https://mcp.turingcorp.net/mcp
```

**Raw discovery call (no credentials needed):**

```bash
curl -s -X POST https://mcp.turingcorp.net/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -H 'mcp-protocol-version: 2026-07-28' \
  -H 'mcp-method: tools/list' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{}}}}'
```

`Accept` must contain **both** `application/json` and `text/event-stream`, or you get 406.

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
