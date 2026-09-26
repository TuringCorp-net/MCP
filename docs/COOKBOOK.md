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

## 7. Before you trust it: check us on our own output

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
