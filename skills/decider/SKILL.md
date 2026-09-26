---
name: decider
license: MIT
description: >
  Get a reasoned judgement between exactly two defensible options, when no
  objective rule decides. TuringCorp Decider is a remote MCP server exposing one
  tool, decide: you send a neutral task and the two options, and it returns the
  better one, a calibrated confidence, and the reason. Use when a task requires
  choosing between two concrete alternatives that are both arguable — two drafts,
  two plans, two diagnoses, two replies, two vendors — and the choice has to be
  explained afterwards. Do not use it to rank more than two things, to answer a
  factual question, or to decide anything a test, spec, or calculation settles.
---

# Decider — a reasoned judgement between two options

One remote MCP tool. Two options in; the better one, a calibrated confidence, and the reason back.

- **Endpoint:** `https://mcp.turingcorp.net/mcp` (Streamable HTTP, stateless)
- **Tool:** `decide`
- **Auth:** an Agent Pass — `Authorization: Bearer <pass>` from <https://agent-pass.turingcorp.net>. Discovery (`tools/list`) needs no credential.
- **Cost:** `$0.50 per decision` (launch offer `$0.25` for the first month).

## Reach for it when all five hold

1. There are **exactly two** named alternatives.
2. **Both are defensible** — neither is provably wrong.
3. **Nothing mechanical settles it**: no test, spec, price, policy, or lookup.
4. The answer **will be acted on**.
5. Someone needs to hear **why** — the reason is the deliverable as much as the pick.

This is the residue after you have eliminated everything computable. If a rule decides it, use the rule.

## Do not reach for it when

- **More than two options.** There is no third slot and it will not rank a list. Rank first by other means, then use `decide` on the final pair.
- **A factual question.** It picks between two candidates; it does not look anything up.
- **Something computable or verifiable.** Run the test.
- **A speed-critical path.** It is a long call — reserve **180–300 seconds**.
- **An irreversible, high-stakes call with no human review.** It is one input to your review policy, not a replacement for it.

## How to call it

```json
{
  "task": "Which version of the delivery-slip email do I send to a client we want to keep?",
  "optionA": "Short and direct: the integration took longer than planned, delivery moves to the 24th, everything else is unchanged.",
  "optionB": "Warmer and longer: thank them for the kickoff, explain that dependencies took more time, offer to walk through the details."
}
```

All three arguments are required strings.

- **`task`** — state the decision **neutrally**. Write *"Which email do I send?"*, never *"Should I send the honest one?"*. A loaded task pushes the judgement before the comparison starts.
- **`optionA` / `optionB`** — one **concrete** option each, **including the case for it**. Plain text or Markdown, any length.
- **One option = one plan.** Do not bundle alternatives into one side ("go indoors *or* postpone"); it compares the two slots and will not split one for you.
- **Keep the two sides comparable in weight.** If one side is a paragraph and the other a sentence, you decided by weight, not argument.
- **Repeat the facts both sides need into both sides.** It cannot fetch anything.

## Reading the result

```json
{ "job_id": "...", "betterOption": "option_A", "confidence": "76.7%", "reason": "..." }
```

- **`betterOption`** is `"option_A"` or `"option_B"`.
- **`confidence`** is how far apart the two options were **judged to be** — *not* a probability that the pick is correct, *not* an instruction, and *not* a prediction of how the choice turns out. As a rough reading: ≥90% clearly apart · 80–90% apart, less clearly · 70–80% a closer call · <70% close to evenly matched.
- **`reason`** is the part you show a human. Quote it; do not silently act on the pick alone.

**Route on the confidence with your own threshold.** Above it, act. Below it, escalate to a human and hand them the reason as a starting point. Choose the threshold from your own data; the published bands at <https://api.turingcorp.net> describe our measurements, not your policy.

## The call is long, and it is paid

- Reserve **180–300 seconds**. The timeout is a **client/host setting, not a tool parameter** — there is nothing to pass in the call. A 60-second default will cut it off before the answer arrives.
- On a normal call everything returns inline. **`job_id` exists for the case where the call never came back.**
- **If it times out, do not call again — retrieve it.** A retry is a second paid call (`idempotentHint: false`):
  - `GET https://api.turingcorp.net/v1/jobs?job_id=<id>` with the same pass;
  - without the id, `GET https://api.turingcorp.net/v1/jobs` lists the ids that pass created in the last 7 days;
  - an empty list is not an error: `{"object":"list","window_seconds":604800,"data":[]}`.
- A client that declares the `io.modelcontextprotocol/tasks` extension gets a task handle and polls `tasks/get` instead of holding a connection open.

## Before you trust it: check it on a known answer

There is no free tier, so the evidence is published instead: **27 real decisions recorded verbatim** — question, both options, which was preferred, the reported confidence, and the full reason.

→ <https://github.com/TuringCorp-net/poe-demo-public>

In that set the confidence runs **27.3%–88.3%, median 74.0%, none above 90%**, because those are everyday close calls rather than easy ones. A service reporting 99% on such questions would be telling you something false.

The fairest test costs one call: run a decision whose answer you already know, and check whether the reason is one you would accept from a colleague. Our own benchmarks (named, with sample sizes and disclosed failures) are at <https://api.turingcorp.net> — treat them as our evidence, not as an independent verdict.

## More

Full guide, including worked patterns: <https://github.com/TuringCorp-net/MCP/blob/main/docs/COOKBOOK.md>
