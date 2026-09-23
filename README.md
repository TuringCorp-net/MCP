# TuringCorp — MCP server

A **judge for agent decisions**. Your agent has two defensible options and has to pick one. Send both: a **panel of models** judges them together and returns the better one, **how far apart it judged them** (a calibrated confidence), and why.

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
| **Out** | `job_id` (string), `better_option` (`"A"` \| `"B"`), `confidence` (percentage string, e.g. `"83.3%"`), `reason` (string) |
| **Annotations** | `readOnlyHint: true` · `openWorldHint: false` · `idempotentHint: false` |
| **Latency** | Expect 60-90 seconds per call: set your client timeout to at least 180 seconds (300 recommended). A 60-second default cuts the call off before the answer arrives. |
| **Long calls** | A client that declares the `io.modelcontextprotocol/tasks` extension gets a task handle back instead of waiting 60-90 seconds, and polls `tasks/get` — see [Retrieving a result](#retrieving-a-result). Clients that do not declare it see no change at all. |

`confidence` is **this service's own judgement of how far apart the two options were** — a reference for your decision-making, not an instruction, not a result, and not a prediction of how the choice turns out. As a rough reading: ≥90% clearly apart · 80–90% apart, less clearly · 70–80% a closer call · <70% close to evenly matched. These ranges are descriptive only. **Choose your own threshold for your own use case; for high-stakes or irreversible decisions apply your own review policy.** Observed accuracy for each range, and how it was measured, is published at <https://api.turingcorp.net> — that page is the authority on the bands and their measured accuracy.

⚠️ `idempotentHint: false` is an honest declaration: a retried call is a new call. **Record the `job_id` the result returns** — a decision you have already paid for can be collected for **7 days** with the same credential (`GET https://api.turingcorp.net/v1/jobs?job_id=<id>`; without the id, `GET https://api.turingcorp.net/v1/jobs` lists the job ids for that credential). Retrieve instead of retrying.

## Retrieving a result

Every call returns a `job_id`. If the call times out or the connection drops, **do not call again** — retrieve it:

- **With the id** — `GET https://api.turingcorp.net/v1/jobs?job_id=<id>` with the same Agent Pass. A client that speaks the `io.modelcontextprotocol/tasks` extension can use `tasks/get` with `{"taskId":"<id>"}` on the MCP endpoint instead.
- **Without it** — `GET https://api.turingcorp.net/v1/jobs` lists the job ids that credential created in the last 7 days; then fetch one as above.

Retrieval returns the job's status and, once it succeeded, the same body the call itself would have returned. A job that is not yours, or older than **7 days**, is reported as unavailable. `GET https://api.turingcorp.net/v1/account` returns `{"account_id":"…"}`.

## See it decide first

There is no free tier, so we publish the evidence instead: **27 real decisions, recorded verbatim** — the question, both options, which one was preferred, the confidence reported, and the reason. Nine domains, three cases each: Writing · Tech · Business · Research · Career · Money · People · Travel · Everyday.

→ **<https://github.com/TuringCorp-net/poe-demo-public>** — dataset: [`examples.json`](https://github.com/TuringCorp-net/poe-demo-public/blob/main/examples.json)

Read the confidence column first. On ordinary, closely matched questions it reports **70–90%, not 99%** — and that is the point of a calibrated number rather than a decorative one: a high value means the comparison was decisive and the pick can be acted on, a low value means the two options really are close and the choice stays yours. In this set the range is **27.3%–88.3%, median 74.0%, none above 90%**, because these are everyday close calls, not easy ones. What the bands deliver when they *are* decisive is published at <https://api.turingcorp.net>.

## Authentication

Send an **Agent Pass**: `Authorization: Bearer <pass>`.

Issued at **<https://agent-pass.turingcorp.net>** — self-service signup (email verification) → top up → get a pass. A pass is valid for **7 days** and can be re-rolled. If a call is refused as an invalid credential, sign in there again and re-roll.

Errors fall into **two classes**, and they arrive in different places:

> Credential problems are rejected before the call (HTTP 401 + `WWW-Authenticate`). Business failures come back as a tool result with `isError: true` plus a JSON block `{error, http_status, action_url, message}` — `http_status` is the upstream status; the tool call itself is HTTP 200.

| Class | What comes back | Examples |
|---|---|---|
| **Credential** | HTTP `401` + `WWW-Authenticate` — no tool result is produced | no credential · expired or invalid pass (`invalid_credential`, with a re-login URL in the message) |
| **Business** | HTTP `200`, tool result `isError: true`, plus a JSON block | insufficient balance (`http_status` `402`, `action_url` pointing at top-up) · over quota (`http_status` `429`) |

Branch on the **HTTP status first**: a `401` will not succeed on retry without a new credential, whereas a business failure was a call that ran and was declined — read the block for the reason and the next step. The detailed contract is under *Errors are machine-readable* below.

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

### Versions

Two version numbers exist and they are **not** meant to match:

| | meaning |
|---|---|
| **Registry `version`** (`server.json`) | the **listing** version - the metadata record (endpoint, auth declaration, icon, links). Advances when the listing changes. |
| **`serverInfo.version`** (from the live endpoint) | the **runtime** version of the deployed server. Advances when behavior changes. |

A listing-only change (for example a corrected link) advances the registry version without
touching the runtime, so the two can legitimately differ.

### Protocol details worth knowing

- **`Accept` must contain both** `application/json` **and** `text/event-stream`, or you get 406.
- **Protocol versions:** one route serves both generations.
  - **Modern `2026-07-28`** — no `initialize` handshake; use `server/discover` and `_meta`.
  - **Legacy** — standard `initialize` handshake. Measured negotiation: a client asking for `2025-11-25`, `2025-06-18`, or `2025-03-26` is answered with **exactly the version it asked for**; a modern `2026-07-28` `initialize` (which is not part of that generation) is answered with `2025-11-25`.
  - Server capability discovery via `tools/list` and `server/discover` needs no credentials.
- **Errors are machine-readable.** There are two distinct kinds, and they arrive differently — so branch on the right thing:

| What failed | How you see it | What to branch on |
|---|---|---|
| **Credential** missing or invalid | **HTTP `401`** with a `WWW-Authenticate` challenge (carrying `error="invalid_token"` and `resource_metadata`); the body is a standard JSON-RPC error with `error.code = -32001`. No tool result is produced. | the **HTTP status code** |
| **Business** failure — insufficient balance, quota, upstream error | a normal tool result (`isError: true`) whose **second `content` block** is a single JSON object you can `JSON.parse`, e.g. `{"error":"insufficient_balance","http_status":"402","action_url":"…","message":"…"}` | the **`error` field** inside that block |

  In short: credential problems are rejected before the tool runs, so they never appear as a tool result; everything else that fails after that does, with a machine-readable block attached.
- **Authorization discovery:** `401` responses carry a `resource_metadata` pointing at `/.well-known/oauth-protected-resource`. Note that this server uses a **static bearer credential, not OAuth** — that document says so explicitly rather than sending you into an OAuth flow that does not exist.

## Not for browser clients

This endpoint deliberately does **not** serve browser-based (cross-origin) MCP clients: requests carrying a browser `Origin` header other than localhost are rejected with 403, and no CORS headers are returned. Browser tooling such as the MCP Inspector works by proxying through a **local** process, which sends no `Origin` — that path works normally. If you need a browser page to reach this server, run a proxy you control rather than calling it cross-origin.

## What this is not

- **One tool only.** `decide` is the entire surface. There is no way to reach other tiers through this endpoint.
- **No SLA.** No availability commitment is offered, and none should be inferred.
- **Not an autopilot.** Decider is a component you call. How you gate on it — thresholds, human review, retries — is your policy and stays yours.
- **No idempotency key — record the job id instead.** A retried call is a new call; collect the result with the `job_id` instead.

## About this repository

This repository carries the **discovery metadata** for the server above — what directory services and clients read to find and describe it.

- [`server.json`](server.json) — the official MCP Registry manifest
- [`docs/DIRECTORIES.md`](docs/DIRECTORIES.md) — where this server is listed, and what each directory requires

This server is also listed on Smithery: **<https://smithery.ai/servers/turingcorp/mcp>**

The server itself is a hosted remote endpoint; nothing here needs to be installed or run.

## License

See [LICENSE](LICENSE).
