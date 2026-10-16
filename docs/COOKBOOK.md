# Cookbook: what to hand `decide`

`decide` compares exactly two options and returns the better one, a calibrated confidence, and the reason.
This page is about the part that is actually hard: **recognising the calls it is for, and writing the two
options so the comparison is fair.**

If you read nothing else: **it is for choices where no objective rule decides.** A test, a spec, a price, a
policy, a linter — if something mechanical settles it, use that instead. `decide` is for the residue: two
options that are both defensible and neither is provably right.

---

## 1. The shape of a good call

| | |
|---|---|
| **One decision** | Not "which of these five", not "rank these", not "is this good". Exactly two named alternatives. |
| **Both defensible** | If one option is obviously wrong, you do not need a decision model — you need to delete it. |
| **Nothing mechanical settles it** | No spec, no test, no arithmetic, no lookup. |
| **You will act on the answer** | A decision you cannot use is a decision you should not buy. |
| **The reason matters to someone** | You have to explain the choice afterwards — to a client, a teammate, or yourself next month. |

## 2. Pattern: choose the version you send

The most common shape. You have already written both versions; you cannot tell which is better because you
wrote both.

```json
{
  "task": "Which version of the delivery-slip email do I send to a client we want to keep?",
  "optionA": "Short and direct: the integration took longer than planned, delivery moves to the 24th, everything else is unchanged.",
  "optionB": "Warmer and longer: thank them for the kickoff, explain that dependencies took more time, offer to walk through the details."
}
```

Same pattern for: two diagnoses, two refactor plans, two offer wordings, two vendor replies, two names.

## 3. Pattern: act above a threshold, escalate below it

This is what the confidence is *for*. It is a reading of **how far apart the two options were judged** — not a
probability that the pick is correct, and not an instruction to act.

```
call decide
  → confidence >= your threshold   → act on betterOption
  → confidence <  your threshold   → a human decides; the reason is your starting point
```

Choose the threshold from your own data, not from this page. What the bands actually delivered, and how it was
measured, is published at <https://api.turingcorp.net> — **that page is the authority on the bands**, and it is
the only place we restate the numbers, so there is exactly one version of them to keep correct.

## 4. Pattern: a second opinion before an irreversible step

For a call you cannot undo — a sent message, a signed contract, a published post — the value is not speed. It
is having one more reading before you commit, and a written reason you can point at later.

⚠️ **This is not a substitute for your own review policy.** For high-stakes or irreversible decisions, keep
your own human review; treat the output as one input to it.

## 5. Writing the two options well (the part that decides the quality)

- **`task` — state the decision neutrally.** *"Which email do I send?"*, not *"Should I send the honest one?"*.
  A loaded task pushes the judgement before the comparison starts.
- **One option = one plan.** Do not bundle alternatives into one side ("go indoors *or* postpone"). It compares
  the two slots; it does not split one of them for you.
- **Give the case for each side, not just the label.** Both options should carry the reasoning that makes them
  defensible — otherwise you are asking it to choose between two nouns.
- **Keep the two sides roughly comparable.** If one side is a paragraph and the other a sentence, you have
  decided by weight, not by argument.
- **Put the facts both sides need into both sides.** It cannot look anything up.

## 6. When not to reach for it

- **More than two options.** There is no third slot; it will not rank a list.
- **Anything you can compute or verify.** Use the test, the spec, the price.
- **Factual questions.** It picks between two candidates; it does not look anything up.
- **Speed-critical paths.** A decision is a long call and a paid one — do not put it behind a request that has
  to answer in seconds. Reserve 180–300 seconds.
- **High-stakes irreversible calls without review.** See §4.

## 7. If a call is cut off: retrieve it with `get_result`

**Do not call `decide` again** — a retry is a second paid call.

**Retrieve it with the `get_result` tool.** This matters more than it sounds: retrieval needs the Agent Pass, and
inside a host the **agent does not have it**. So retrieval is deliberately a *tool call*, not a REST call — the host
attaches the credential, and **no credential is ever a tool argument**, because that would put it into prompts and
transcripts.

- **With a `job_id`** → that job's status, and once it succeeded the same decision body the original call returned.
- **With no argument** → the job ids this credential created in the last 7 days. **This is the case that matters:
  when a call is cut off you never received an id.**
- **Read-only and free** (`readOnlyHint: true`, `idempotentHint: true`) — no new work, no charge, safe to repeat.
- A host that declares the `io.modelcontextprotocol/tasks` extension can poll `tasks/get` instead. **Most clients do
  not declare it yet** — which is exactly why `get_result` exists rather than us telling agents to go and find a
  credential.

The operator can fetch the same data over REST (`GET https://api.turingcorp.net/v1/jobs?job_id=<id>`).

**So treat retrieval as the exception handler, not the plan.** The plan is to reserve **180–300 seconds** so the
call finishes and returns inline. A design that relies on retrieval is a design that will lose decisions.

## 8. Before you trust it: check us on our own output

We have no free tier, so we publish evidence instead. **27 real decisions, recorded verbatim** — the question,
both options, which was preferred, the confidence reported, and the full reason, across nine domains.

→ <https://github.com/TuringCorp-net/poe-demo-public> · dataset: [`examples.json`](https://github.com/TuringCorp-net/poe-demo-public/blob/main/examples.json)

Read the confidence column first. On ordinary, closely matched questions it reports **70–90%, not 99%** — in
that set the range is **27.3%–88.3%, median 74.0%, none above 90%**, because those are everyday close calls
rather than easy ones. A model that returned 99% on those would be telling you something false.

**Then check us the way you would check anything else:** run your own two options on a call whose answer you
already know, and see whether the reason is one you would accept from a colleague. That is a fairer test than
any table we could publish, and it costs one call.

The published benchmarks (what they are, sample sizes, what was excluded) are at
<https://api.turingcorp.net>. We name the benchmark, we say they were run by us on the benchmark's own protocol,
and we disclose the failures — so treat them as our evidence, not as an independent verdict.

## 9. Put the call in your test suite: assert on the contract, not the answer

The reason to move a judgement out of a prompt is that the result becomes something you can assert on. The
**shape** of the answer is the contract — a preferred option, how far apart the two were judged, and a reason.
Assert on that, and the same scenario can live in your test suite like any other:

```python
# tests/test_escalation.py
def test_close_call_escalates_to_human(decide):
    r = decide(
        task="Which support reply do we send this customer?",
        optionA="Refund immediately, no questions.",
        optionB="Explain the policy and offer a partial credit.",
    )
    # The pick is not the contract. The shape of the answer is.
    assert r["betterOption"] in ("option_A", "option_B")
    assert r["reason"], "a pick with no reason is not reviewable"

    if float(r["confidence"].rstrip("%")) < 70:
        escalate_to_human(r["reason"])   # your policy, not ours
```

- **Do not assert on which option won.** That re-records one answer, and you would be editing the test every
  model change. Assert that a pick came back, that it carried a reason, and that *your* threshold did its job.
- **The threshold is yours, not ours.** `confidence` reports how far apart the two options were judged; it does
  not tell you what to do about it. That last line is a policy you own and can change without us.
- **The reason is a first-class output**, because it is what you hand a human when the call was close.

The write-up behind this pattern — including the three client-config traps (`"type"`, discovery vs. calling, and
the timeout you have to raise yourself) — is at
<https://dev.to/turingcorp/your-agents-hardest-call-is-the-one-you-cant-test-4cl5>.

## 10. "Connected" is not "credentialed" — test the three states separately

A client panel showing **connected** only means the discovery half answered. Listing is deliberately open, so it
answers whatever you put in the `Authorization` header — including a garbage one. The credential is only exercised
on a call. Probe the three states on the **calling** path:

```bash
# 1) no credential, tools/list -> must be 200. Connectivity, nothing more.
curl -s -o /dev/null -w 'open    %{http_code}\n' -X POST "$MCP_URL" \
  -H 'content-type: application/json' -H 'accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

# 2) a deliberately wrong credential, on a CALL -> must be 401.
#    Do not use tools/list here: it answers 200 whatever the header says.
curl -s -o /dev/null -w 'wrong   %{http_code}\n' -X POST "$MCP_URL" \
  -H 'content-type: application/json' -H 'accept: application/json, text/event-stream' \
  -H 'authorization: Bearer definitely-not-a-real-pass' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"decide","arguments":{}}}'

# 3) the real credential, same call -> the only line that proves the setup works end to end.
curl -s -o /dev/null -w 'real    %{http_code}\n' -X POST "$MCP_URL" \
  -H 'content-type: application/json' -H 'accept: application/json, text/event-stream' \
  -H "authorization: Bearer $AGENT_PASS" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"decide","arguments":{}}}'
```

Measured against this server (2026-10-10): state 1 → `200`, state 2 → `401` with a JSON-RPC credential error,
state 3 → the decision. If state 2 returns `200`, the calling path is not protected — that is a finding about the
server, not about your config.

| Symptom | Most likely cause | First thing to check |
|---|---|---|
| Panel says connected, first call 401 | credential not attached by the client config | the `headers` block, **including the `Bearer ` prefix** |
| `405` right after adding the server | transport dialect mismatch (SSE fallback) | the `type` field for that specific client |
| Server entry missing from the client UI | wrong config key (`servers` vs `mcpServers`) | the JSON key, not the URL |
| Call dies at ~60 s, no error body | client tool-call timeout, not a server fault | timeout unit and value; raise it to 180–300 s |
| Works with `curl`, fails in the host | header bridge (`mcp-remote` and friends) ignores env vars | move the header into `--header` |
| 401 on *retrieval* of an earlier call | retrieval attempted over REST without a host-attached credential | use the retrieval tool call instead |

The per-client config matrix behind that table is `llms-install.md` in this repository; the prose version of this
section is at
<https://dev.to/turingcorp/your-remote-mcp-server-is-connected-and-still-returns-401-333g>.

## 11. Client config matrix — the same transport, four different names

Every host names the streamable-HTTP transport differently, and a wrong name fails in three different ways.
Two of them do not look like configuration errors: a bare `url` with no `type` can be read as a **local stdio**
server (you get a spawn error for an endpoint that is never dialed), and a wrong/absent `type` in Cline makes it
fall back to **SSE**, which this server does not speak, so the connection dies with a **`405`** that reads like a
server fault.

| Client | Config location | Parent key | Transport field | Value |
|---|---|---|---|---|
| Claude Code | CLI | — | `--transport` | `http` |
| Cursor / Windsurf / Claude Desktop | JSON settings | `mcpServers` | `type` | `http` |
| Cline | `cline_mcp_settings.json` | `mcpServers` | `type` | `streamableHttp` |
| VS Code | `.vscode/mcp.json` | `servers` | `type` | `http` |
| Codex | `~/.codex/config.toml` | `[mcp_servers.<name>]` | `url` + `http_headers` | table keys |
| stdio-only hosts | JSON settings | `mcpServers` | `command` + `args` | `npx mcp-remote` |

The one that catches people: **Cline wants `"streamableHttp"`** (camelCase, no hyphen). Anything else, or omitting
`type`, is an SSE fallback.

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

For a stdio-only host, bridge with `mcp-remote` and pass the credential through `--header` — the bridge does
**not** read an `AUTHORIZATION` environment variable, so an env-only config connects, lists tools, then fails on
the first call with `401`. The missing space after `Authorization:` is deliberate: some hosts mangle spaces
inside `args`.

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

Per-client starting points, including the `.vscode/mcp.json` and Codex TOML shapes, are in `llms-install.md` in
this repository. The write-up behind this section — including a diagnostic you can run before opening a client
at all — is at
<https://dev.to/turingcorp/the-remote-mcp-client-config-matrix-nobody-documents-and-the-three-ways-type-fails-silently-1chg>.

## 12. When a call is cut off: four failures, one recovery path

A cut-off call is not one event. It arrives as four different signatures, and only one of them means "nothing
happened". The signature decides whether retrying is safe, and for a paid non-idempotent call the answer is
almost always **retrieve, don't retry**.

| What the caller sees | Did the job run? | Retry safe? | Move |
|---|---|---|---|
| Client aborts the call at its own limit | Usually yes | No, until you look | Retrieve by id, or list the ids your credential created |
| Error result (`isError: true`) | Rejected before or during work | Depends on the block | Branch on the HTTP status in the error block |
| Framework backgrounds the call, hands back a task id | Yes | No | Wait for the task notification, or retrieve by id |
| Idle abort (no bytes for the idle window) | Usually yes | No | Retrieve by id |

**The number you are fighting is usually not the wall-clock timeout.** On an HTTP connection the per-request timer
stops at the first response byte, so a tool that thinks for minutes and then answers is racing the **idle** window
(Claude Code: five minutes for HTTP/SSE, configurable, `0` disables). A per-server `timeout` of at least 1000 ms
also acts as a floor on that idle window — so the effective setting for a long call is not the largest number you
can find, it is the per-server timeout plus the knowledge that `decide` sends nothing until it answers.

**Retrieval is a tool call, not a REST call, from inside a host.** The host holds the Agent Pass and attaches it;
an agent that reaches for the REST API itself gets `401` on a job it is allowed to read. `get_result` is read-only
and free — safe to repeat — and with no argument it lists the ids your credential created in the last 7 days,
which is the case that matters, because a client-side abort never gave you an id.

**Test the retrieval path before you need it.** Call `get_result` with no argument while nothing is broken: a
`401` means retrieval is not reachable from where the agent runs and your recovery plan is a diagram, not a
capability; an empty list is the healthy state before the first cut-off.

Two contract rules worth copying if you build the server side: answer **"not yours" and "does not exist"
identically**, or the endpoint becomes an oracle for probing other tenants' ids; and return an **empty list as a
normal result** with no "id 0" sentinel, because every client eventually probes it — `job_id=0` is answered as an
unavailable job, not as a placeholder.

Per-client timeout controls, where the client documents them:

| Client | Per tool call | Idle window |
|---|---|---|
| Claude Code | per-server `timeout` (ms) in `.mcp.json`, or `MCP_TOOL_TIMEOUT`; unset ≈ 28 h | 5 min for HTTP/SSE, `CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT` |
| Cline | `timeout` in `cline_mcp_settings.json`, **seconds** | — |
| Cursor / Windsurf / VS Code | not documented in their MCP pages — treat the default as unknown | — |

The full write-up, including the negative test as a table and the five design rules for a retrievable long call,
is at <https://dev.to/turingcorp/a-tool-call-that-gets-cut-off-is-four-different-failures-and-only-one-of-them-is-a-timeout-2532>.

## 13. A reported confidence is not a measurement

A model will happily append `"confidence": 0.92` to its answer, and the obvious next line is a branch on it.
That number was generated in the same forward pass as the answer, so it tracks how decisive the answer *sounds*,
not how close the evidence was. It is a word that looks like a number.

A **measured** number is produced by bookkeeping: store the number now, store what actually happened later, and put
the two in the same table. That join is the only place calibration can come from, and it is why "ask the model to
report a better-calibrated number" cannot work — re-wording moves the numbers, not the missing column.

| | Decorative number | Measured number |
|---|---|---|
| Where it comes from | a sampled token in the same generation as the answer | a record: number stored now, outcome stored later, buckets compared |
| What it responds to | how decisive the answer sounds | how close the evidence actually was |
| Behaviour on a harder input | nearly unchanged | falls, often below 0.7 |
| What you can compute from it | nothing — no join is possible | per-band hit rate, and whether the bands are honest |
| Failure mode when you branch on it | a coin flip with a confident label, invisible in production | you own the errors, but you can see them and fix the bands |
| Cheapest way to tell them apart | ask the same question twice with different framing; watch the number track the framing | ask whether your logs contain the outcome column at all |

Our own record is published: on 27 verbatim decisions where both options were defensible, the reported separation
runs **27.3%–88.3%, median 74.0%, none above 90%** — a checked number on a genuinely close call is *allowed* to be
low, so it is.

The fix is not a better prompt; it is making the judgement a call with a contract, under an id you can retrieve:

```bash
curl -s -X POST "https://mcp.turingcorp.net/mcp" \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -H "authorization: Bearer $AGENT_PASS" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"decide","arguments":{
        "task":"Which draft of the outage notice do we send to customers?",
        "optionA":"Short: what broke, what we did, when it was fixed.",
        "optionB":"Longer: the same content plus what we changed so it cannot recur."
      }}}'
```

The result carries a `job_id` next to the pick and the reason — treat it as the first column of your bookkeeping,
not as error handling. Then two moves, both boring and both mandatory: **log every judgement as a row** (id, number,
inputs, timestamp, action taken), and **join the rows to outcomes later**, bucketed by band.

Checklist to paste into the review that gates the prompt change:

1. Do not branch on a number whose per-band accuracy you have never computed.
2. Log every judgement: id, number, inputs, timestamp, and the action taken.
3. Define the bands **before** looking at outcomes, or you will fit the bands to the data and call it calibration.
4. Join the log to outcomes; compute the hit rate per band; keep the table where the team can see it.
5. When the numbers and the outcomes disagree, change the band-to-action mapping — not the prompt wording.

The full write-up is at
<https://dev.to/turingcorp/a-confidence-number-your-model-reports-is-not-a-measurement-3g94>.

## 14. A tool description is a protocol, not documentation

When a model decides whether to call a tool, the only things in its context are the name, the description, the
parameter descriptions and the schemas. The README, the docs site and the comment block never travel with the call.
So the description is not documentation about the tool — it is the tool's interface to the model.

The expensive failures are the quiet ones: the tool that is never called because nothing said *when* to call it, the
call made on input the tool cannot serve because nothing said what it is **not** for, the retry after a timeout
because the description never mentioned that a retry is a new call.

**Seven questions to answer in the description, per tool.** Every "no" is a decision you have not made yet:

| # | Question | Where it lives |
|---|---|---|
| 1 | When should the model call this, named as a task rather than a feature? | first sentence of the description |
| 2 | When should it **not** — including the nearest lookalike task you do not serve? | description |
| 3 | Can every parameter be filled correctly without reading another document? | parameter descriptions |
| 4 | What comes back, which fields are always present, which are conditional? | output schema |
| 5 | How long may the call take, and does calling again cost something? | description |
| 6 | After a cut-off, what is the path that is **not** "call it again"? | description, naming the other tool |
| 7 | Are the declared annotations true (`readOnlyHint`, `idempotentHint`, `openWorldHint`)? | annotations |

Read the contract the model actually sees over the same credential-free discovery call its host makes:

```bash
curl -s -X POST https://mcp.turingcorp.net/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Our own description is published the same way, unauthenticated: `tools/list` and `/llms.txt`. The full write-up is at
<https://dev.to/turingcorp/your-tool-description-is-a-protocol-not-documentation-25f2>.
