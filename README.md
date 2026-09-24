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
| **In** | `task` (string), `optionA` (string), `optionB` (string) — all required |
| **Out** | `job_id` (string), `betterOption` (`"A"` \| `"B"`), `confidence` (percentage string, e.g. `"76.7%"`), `reason` (string) |
| **Annotations** | `readOnlyHint: true` · `openWorldHint: false` · `idempotentHint: false` |
| **Timeout** | Reserve **180–300 seconds** — a decision is a long call. The timeout is a **client/host setting, not a tool parameter**: there is nothing to pass in the call. A 60s default cuts it off before the answer arrives; if that happens, do not call again — retrieve by `job_id`. |
| **Long calls** | A client that declares the `io.modelcontextprotocol/tasks` extension gets a task handle back instead of holding one connection open for minutes, and polls `tasks/get` — see [Retrieving a result](#retrieving-a-result). Clients that do not declare it see no change at all. |

**A successful call returns the decision inline.** `job_id`, `betterOption`, `confidence` and `reason` all arrive
in the *same* tool result — there is nothing to poll and nothing to fetch afterwards. The `job_id` is only for the
case where the call never came back (timeout, dropped connection, client gave up waiting).

`confidence` is **this service's own judgement of how far apart the two options were** — a reference for your decision-making, not an instruction, not a result, and not a prediction of how the choice turns out. As a rough reading: ≥90% clearly apart · 80–90% apart, less clearly · 70–80% a closer call · <70% close to evenly matched. These ranges are descriptive only. **Choose your own threshold for your own use case; for high-stakes or irreversible decisions apply your own review policy.** Observed accuracy for each range, and how it was measured, is published at <https://api.turingcorp.net> — that page is the authority on the bands and their measured accuracy.

⚠️ `idempotentHint: false` is an honest declaration: a retried call is a new call. **Record the `job_id` the result returns** — a decision you have already paid for can be collected for **7 days** with the same credential (`GET https://api.turingcorp.net/v1/jobs?job_id=<id>`; without the id, `GET https://api.turingcorp.net/v1/jobs` lists the job ids for that credential). Retrieve instead of retrying.

## When not to use it

- **More than two options.** It compares exactly A and B — there is no third slot, and it will not rank a list.
- **Anything you can compute or verify.** A spec, a test, a price, a document: if something objective decides it, use that. This is for choices where no objective rule does.
- **Factual questions.** It picks between two candidates; it does not look anything up.
- **Speed-critical paths.** A decision is a long call and a paid one. Do not put it behind a request that has to answer in seconds.
- **High-stakes irreversible calls without review.** Route on the confidence and keep your own review policy.

## How to fill the three arguments

- `task` — state the decision **neutrally**, without leaning toward either side: *"Which email do I send?"*, not *"Should I send the honest one?"*
- `optionA` / `optionB` — one **concrete** option each, plus the case for it. Plain text or Markdown, any length; keep the two sides roughly comparable so the comparison is fair.
- **One option = one plan.** Do not bundle alternatives into a single side ("go indoors *or* postpone"): it compares the two slots, it does not split one of them for you.

```json
{
  "task": "Which version of the delivery-slip email do I send to a client we want to keep?",
  "optionA": "Short and direct: the integration took longer than planned, delivery moves to the 24th, everything else is unchanged.",
  "optionB": "Warmer and longer: thank them for the kickoff, explain that dependencies took more time, offer to walk through the details."
}
```

## Retrieving a result

Every call returns a `job_id`. If the call times out or the connection drops, **do not call again** — retrieve it:

- **With the id** — `GET https://api.turingcorp.net/v1/jobs?job_id=<id>` with the same Agent Pass. A client that speaks the `io.modelcontextprotocol/tasks` extension can use `tasks/get` with `{"taskId":"<id>"}` on the MCP endpoint instead.
- **Without it** — `GET https://api.turingcorp.net/v1/jobs` lists the job ids that credential created in the last 7 days; then fetch one as above. The list carries only the job id, the product and a timestamp — fetch a job by id to see what it was.
- **Nothing yet** — an empty list is **not** an error, and there is **no** `job_id=0` placeholder: `{"object":"list","window_seconds":604800,"data":[]}`. Asking for `0` returns `404 No such job.`

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

## Two ways in — and they are not the same

- **Through an MCP client or host** (Claude Code, Cursor, VS Code, Codex, TRAE, Coze…). The host holds the credential and attaches it for you: **you do not set an `Authorization` header yourself, and in most hosts you cannot**. The timeout is a host setting, not something you pass in the call.
- **Directly against the REST API** (<https://api.turingcorp.net>). Here you *do* send `Authorization: Bearer <Agent Pass>` yourself, and you can retrieve a job by id.

⚠️ The part that catches people out: **being able to call the tool through a host does not mean you can reach the
REST API.** The host may never hand you the underlying Agent Pass, so a job-retrieval call you make on your own can
come back `401`. If your host declares the Tasks extension, retrieve through the tool surface instead (see
[Retrieving a result](#retrieving-a-result)); if it does not, retrieval may simply not be reachable from where you are.

## Quickstart

You need an **Agent Pass** to call the tool. Discovery (`tools/list`) works without one — calling `decide` does not.

🔑 **Where the pass lives matters.** If your client can keep it out of the file, do that — VS Code's
`${input:…}`, Codex's `bearer_token_env_var`. A pass written in plain text inside `mcp.json` or any client config
is readable by **every agent and process that can read that file**, and assistants do read their own config — one
can print your pass straight into its output. Treat a client config as public within your machine.

### 1. Get an Agent Pass

Self-service at **<https://agent-pass.turingcorp.net>**: sign up with an email address, verify it, add credit. The pass is shown once; it is valid for **7 days** and can be re-rolled at any time. Agents can do the same thing over the API — see that site's `/llms.txt`.

### 2. Point your client at the endpoint

**Any client that reads a JSON config — Cursor, Cline, Windsurf, Claude Desktop:**

```json
{
  "mcpServers": {
    "TuringCorp": {
      "type": "http",
      "url": "https://mcp.turingcorp.net/mcp",
      "headers": { "Authorization": "Bearer <your Agent Pass>" }
    }
  }
}
```

Keep the `"type": "http"` line: a client that reads a `url` entry with no `type` treats it as a local stdio
server and skips it.

**A client that only speaks stdio** needs a bridge, and the credential must go in through `--header` —
`mcp-remote` does not read an `AUTHORIZATION` environment variable, so a config that only sets one connects,
lists tools, and then fails on the first call with 401. The missing space after `Authorization:` is deliberate:
some clients mangle spaces inside `args`.

```json
{
  "mcpServers": {
    "TuringCorp": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.turingcorp.net/mcp",
               "--header", "Authorization:${AGENT_PASS}"],
      "env": { "AGENT_PASS": "Bearer <your Agent Pass>" }
    }
  }
}
```

> ⚠️ A bare `npx -y mcp-remote@latest https://mcp.turingcorp.net/mcp` with no credential **will connect and list tools, then fail on the first call** with 401. That is expected — discovery is open, execution is not.

**Claude Code:**

```bash
claude mcp add --transport http TuringCorp https://mcp.turingcorp.net/mcp \
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
  --data '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"decide","arguments":{"task":"Pick a launch date","optionA":"Ship now","optionB":"Wait two weeks"},"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{}}}}'
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
