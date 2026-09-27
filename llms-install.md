# llms-install.md — installing the TuringCorp MCP server

Machine-readable setup notes for agents (Cline, Claude Code, Cursor, Windsurf, VS Code, Codex, Coze…).
**Human-facing quickstart: [`README.md`](README.md).** This file only carries what an agent needs to get the
server connected with no guessing.

## What this is

A **remote** MCP server. Nothing is installed, cloned, or run locally — there is no package, no container, no
stdio process. Setup is exactly two things: **a URL**, and **a credential the client attaches for you**.

- **Endpoint (canonical):** `https://mcp.turingcorp.net/mcp`
- **Aliases:** `https://mcp.turingcorp.net/` and `https://mcp.turingcorp.net/mcp/` (same route)
- **Transport:** Streamable HTTP, **stateless**. `GET /mcp` → 405 is normal, not an error.
- **One tool:** `decide`.

## Step 1 — the credential

The endpoint requires an **Agent Pass** (a static bearer token). It is self-service:

1. Go to <https://agent-pass.turingcorp.net>
2. Sign up (email verification), then top up.
3. Copy the pass — it looks like `tuc_…` / `ap_…` and is **valid for 7 days**. If a call is ever refused as an
   invalid credential, sign in again and re-roll it.

**Discovery needs no credential.** `tools/list` and `server/discover` are open, so you can verify the server is
reachable *before* asking anyone for a token.

## Step 2 — configure the client

The credential goes in an `Authorization` header, **including the word `Bearer` and a space**.

**Any client that reads a JSON config** (Cursor, Windsurf, Claude Desktop) — note the key is `mcpServers`:

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

**Cline — read this, the `type` differs.** Cline wants **`"streamableHttp"`** (camelCase, no hyphen) and stores its
settings in `cline_mcp_settings.json`. Any other `type`, or omitting it, makes Cline fall back to **SSE** — and this
server does not speak SSE, so the connection fails with **`405`**:

```json
{
  "mcpServers": {
    "TuringCorp": {
      "type": "streamableHttp",
      "url": "https://mcp.turingcorp.net/mcp",
      "headers": { "Authorization": "Bearer <your Agent Pass>" },
      "disabled": false,
      "autoApprove": [],
      "timeout": 300
    }
  }
}
```

`timeout` is in **seconds** and defaults to 300 — leave it at 300 or higher. Note that Cline has had a bug
([cline#2296](https://github.com/cline/cline/issues/2296), reported on 3.7.0, closed 2025-06-23) where a request
died after roughly a minute regardless of this setting. If a `decide` call is cut off at ~60 s, suspect that class
of bug rather than the server — and **do not retry**: a retry is a second paid call. Retrieve it with the `job_id`
instead.

To open that file in the Cline panel: **MCP Servers** icon (stacked-server icon in the top toolbar) → **Configure**
tab → **Configure MCP Servers**. (The **Remote Servers** tab can add a URL-only server — it picks
`"streamableHttp"` for you — but it has **no field for a custom `Authorization` header**, so this server still
needs the JSON edited by hand to carry the Agent Pass.)

**VS Code** uses the key **`servers`** (not `mcpServers`) in `.vscode/mcp.json`:

```json
{
  "servers": {
    "TuringCorp": {
      "type": "http",
      "url": "https://mcp.turingcorp.net/mcp",
      "headers": { "Authorization": "Bearer <your Agent Pass>" }
    }
  }
}
```

**Claude Code:**

```bash
claude mcp add --transport http TuringCorp https://mcp.turingcorp.net/mcp \
  --header "Authorization: Bearer <your Agent Pass>"
```

**Codex** — edit `~/.codex/config.toml`; the UI has no field for the credential:

```toml
[mcp_servers.TuringCorp]
url = "https://mcp.turingcorp.net/mcp"
http_headers = { Authorization = "Bearer <your Agent Pass>" }
tool_timeout_sec = 300
```

**Clients that only speak stdio** — bridge with `mcp-remote`; headers can *only* come from `--header` (the
`AUTHORIZATION` environment variable is not read):

```json
{
  "mcpServers": {
    "TuringCorp": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "https://mcp.turingcorp.net/mcp",
        "--header", "Authorization: Bearer ${TURINGCORP_AGENT_PASS}"
      ],
      "env": { "TURINGCORP_AGENT_PASS": "<your Agent Pass>" }
    }
  }
}
```

## Step 3 — the one setting people get wrong

**Raise the tool-call timeout to at least 180 seconds; 300 is safer.**

A decision is a long call. The timeout is a **client/host setting — there is nothing to pass in the call.**
Common defaults that are too short: Codex `tool_timeout_sec` = 60 s; TRAE `RUN_MCP_TIMEOUT_MS` = 60000.
Raise them (`tool_timeout_sec = 300`, `RUN_MCP_TIMEOUT_MS = 300000`).

## Step 4 — verify it worked

1. `tools/list` returns one tool, `decide` — **no credential needed**, so this proves connectivity first.
2. Call it once with a real decision.
3. If the call comes back as a credential error, the pass is wrong or expired → re-roll it.

### If a call is cut off by a timeout — read this *before* trying to retrieve it

**Do not call again.** A retry is a second paid call (`idempotentHint: false`).

**But be realistic about getting the first one back.** Retrieval needs the Agent Pass, and **inside an MCP host you
usually do not have it** — the host keeps it in its own config and attaches it for you; it is not in your context.
So the honest picture is:

- **If your host declares the `io.modelcontextprotocol/tasks` extension**, `decide` returns a task handle and you can
  poll `tasks/get` with `{"taskId":"<id>"}`. That path is authenticated by the host and **does** work from inside.
  Verify your host supports it before counting on it — **most clients do not declare it yet.**
- **Otherwise, retrieval is an operator action, not an agent action.** Hand the `job_id` to the human who owns the
  Agent Pass; they run `GET https://api.turingcorp.net/v1/jobs?job_id=<id>` (or list with `GET .../v1/jobs`) and give
  you the result. There is **no** MCP tool for this, by design — the server exposes exactly one tool.

**So the practical answer is not to need retrieval: set the tool-call timeout to 180–300 seconds (Step 3).** A call
that is allowed to finish returns its decision inline, and there is nothing to fetch.

## When to call `decide`

Exactly two defensible options, nothing mechanical decides between them, and someone will need the reason.
Full patterns: [`docs/COOKBOOK.md`](docs/COOKBOOK.md). Agent Skill: [`skills/decider/SKILL.md`](skills/decider/SKILL.md).
