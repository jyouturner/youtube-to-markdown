# What a $0.61 API Call Taught Me About AI Cost Governance

*2026-05-30*

It started with a number that didn't sit right.

I have a small local app — [yt2md](../../README.md) — that turns long YouTube talks into something readable: a digest, a multi-expert "panel" critique, a takeaway, a slide deck. It runs a handful of Claude API calls per video. I was glancing at the usage log and saw the panel call:

> ~4,200 output tokens. Cost: **$0.61**.

Four thousand tokens for sixty-one cents? That felt high. So I asked the dumb question out loud — *how come 4,200 tokens costs that much?* — and started pulling the thread. It did not stop where I expected. By the end I'd found a 3× billing bug in my own code, a "cache" that was pure waste, rebuilt how the app prices itself, added a budget gate, written a governance framework, and stood up a real-time kill switch just to watch it work.

Here's the whole thread, mistakes included.

## The first answer was boring. The second one wasn't.

The boring part: the panel runs on **Opus**, and on Opus *output* tokens cost ~$75/M — 5× the input rate. 4,200 × $75/M ≈ $0.32. Half the cost, explained. "Text-only" in my notes meant *no images*, not *no input* — the call also shipped a big transcript. Fine.

But the arithmetic didn't fully close, so I pulled the actual usage record:

```
input_tokens: 6
output_tokens: 4222
cache_creation_input_tokens: 15732   ← what?
cost_usd: 0.6117
```

Two surprises. First, the input wasn't "a big transcript" sent normally — it was **15,732 tokens written to the prompt cache**, which bills at **1.25× the input rate**. Second, and this is the one that made me stop: when I recomputed the cost from the tokens at the *real* Opus 4.7 rates, I got **~$0.20**, not $0.61.

My own cost meter was wrong by 3×.

## The cost meter was lying

The app had a hardcoded price table. It listed Opus 4.7 at **$15 / $75** per million tokens (input/output). I checked the live pricing page. Opus 4.5 and later are **$5 / $25** — the $15/$75 tier was retired with Opus 4.1. My table had never been updated when the model's price tier dropped.

Every Opus cost in my logs was inflated 3×. The "$0.94/video" I thought I was paying was really closer to ~$0.46. The lesson landed immediately and it's the one I keep coming back to:

> **The thing measuring your cost can be the thing that's wrong.** A stale pricing constant doesn't throw an error. It just quietly misreports reality until you happen to check.

I corrected the table. But "I'll keep this hand-edited constant accurate" is exactly the promise that had already failed once. More on that later.

## The cache that cost money for nothing

Back to that `cache_creation: 15732`. Prompt caching is supposed to *save* money, so why was it a cost line?

Because caching is a bet, not a discount:

| Operation | Price vs normal input |
|---|---|
| Cache **write** | **1.25×** (you pay a 25% *premium*) |
| Cache **read** | **0.10×** (90% off) |

You pay 1.25× once to store something, then 0.10× every time you reuse it. It pays off only if you read it back. My digest, panel, and takeaway calls were each **one-shot** — write the context, get one answer, done. Across 55+ calls of each, the cache was read back exactly **zero** times.

I was paying the 25% storage premium and collecting the reuse discount *never*. The "optimization" was a pure ~$0.03/video loss. Whoever added `cache=True` (me) had the same intuition you probably have — "caching = cheaper" — and never checked that there was a reader.

> **Caching is the wrong tool for a one-shot call.** The API will happily let you write a cache nobody reads and bill you the premium for it.

The fix was deleting three `cache=True` flags. Verified with a live call: cache-creation tokens dropped from ~6K to 0, output byte-identical. Negative-cost change.

## Stop hardcoding prices: ask the invoice

The pricing-table staleness still bothered me. Hand-maintained constants rot. Could I pull prices from an API instead?

Anthropic's Models API returns no prices. But the **Admin Cost API** (`/v1/organizations/cost_report`) returns something better than a rate card — it returns *what you were actually billed*, by model and token type. I wired the app to it and computed effective rates: billed dollars ÷ tokens. The result matched my corrected table to the penny:

```
Sonnet input   $0.9354 / 311,793 tok = $3.00/M   ✓
Opus output    $0.7854 /  31,418 tok = $25.00/M  ✓
```

So I made the table **self-calibrating**: `cost_report ÷ usage_report` derives the real $/Mtok, writes a cache, and the hardcoded table demotes to a fallback + sanity bound (reject any derived rate that diverges >3× — a guard against a parsing bug silently corrupting the numbers). When Anthropic changes prices, the app re-derives them from the next invoice instead of waiting for me to notice.

The staleness bug that started everything can't happen the same way twice.

## From fixing costs to governing them

At this point I had a clean question worth generalizing: **how do you actually control spend on an API-based AI app?** I split it into two buckets — controls *inside* my process, and controls *outside* it — and went looking for what I didn't know. It turned out to be three layers, separated by one hard line: **can it reject a single request, in real time, on a budget basis?**

**Layer 1 — in-app.** Token caps, model routing, a budget gate, `count_tokens` pre-flight pricing, caching done *right*. Fast, free, but single-app and soft. I added a budget gate here: it refuses to *start* a new digest once month-to-date spend crosses a threshold (warn at $15, block at $18), reading authoritative spend from the Cost API. Its blind spot, by construction: it checks at digest start, and billing data lags ~5 minutes, so it can't stop a runaway *mid-pipeline*.

**Layer 3 — provider-native.** Anthropic's Admin API, workspaces, spend caps. This is where I'd assumed the "kill a runaway process" button lived. It doesn't. **The Cost API is reporting, not enforcement** — ~5-minute latency, no per-request rejection. The closest thing to a programmatic kill switch is *disabling an API key*, which stops the *next* request, not the in-flight one. Setting spend or rate limits? **Console-only** — not even exposed in the API. (I confirmed this the tedious way, probing every endpoint.)

**Layer 2 — the gateway you run.** This is the layer I hadn't named, and it's the only one that can hard-reject an individual request in real time. A proxy between your app and the LLM provider, tracking spend per key and returning an error *instead of forwarding* once a budget is blown. Provider-agnostic, too — the same chokepoint can front Anthropic or OpenAI.

> The control you reach for first — the provider's dashboard — is the one that *can't* stop a single runaway request. That capability only exists at a chokepoint you own.

## Proving the kill switch (and finding the trap)

I didn't want to take "a gateway can do real-time cutoffs" on faith, so I stood up [LiteLLM](https://docs.litellm.ai) locally in front of Anthropic, minted a virtual key with a tiny `max_budget`, and hammered it:

```
call 1–4 → HTTP 200 (ok)
call 5   → HTTP 429  budget_exceeded: Current cost: 0.000124, Max budget: 0.0001
```

There it is — rejected **at the gateway, before forwarding**. Proof it never billed upstream: the cost froze at exactly four calls' worth; the blocked 5th added nothing. This is the Layer-2 capability neither the app gate nor the Admin API has.

But the gotchas were as valuable as the success:

1. **My first config was a *false* kill switch.** A DB-free global `max_budget` enforced *nothing* — the proxy forwarded every call, no error, no warning. Budget enforcement requires a stateful Postgres-backed deployment. Anyone who sets a global budget without a database and assumes they're protected has zero protection.
2. **A gateway is infrastructure, not a setting.** Postgres (no SQLite), the Prisma client, `prisma generate` + `db push`, a real venv. That weight *is* the decision: for a single-user tool, it's overkill against a Console spend cap plus an app-side gate. For >1 app, >1 user, or multi-provider, it's exactly right.

## What I'd tell past-me

The specific bugs are almost beside the point. The transferable lessons:

- **Measure, don't assume — and audit the thing that measures.** A $0.61 number I trusted was wrong by 3× because of a constant nobody had touched in a model generation. The meter needs to be checked like any other input.
- **Every "optimization" needs a verified mechanism.** Prompt caching *can* save 90%; mine lost 25% because there was no reader. The feature being real doesn't make your use of it real.
- **Prefer the source of truth over a model of it.** Don't hand-maintain a price table when the provider will tell you what it actually charged. Calibrate from the invoice.
- **Know which layer a control lives in.** Reporting ≠ enforcement. Soft ≠ hard. Real-time per-request rejection is a *gateway* capability — don't expect it from a billing dashboard.
- **Right-size the governance to the blast radius.** The "correct" enterprise answer (a gateway) was the wrong answer for a personal tool. A Console spend cap plus a soft app-side gate was enough. Adopt the heavier layer when scale actually demands it, not because it's more thorough.

The whole thing started because four thousand tokens cost sixty-one cents and that felt off. It's worth letting "that feels off" pull you down the rabbit hole. Mine paid for itself many times over — and the app now knows, to the penny, what it costs.

---

*Full technical writeup, including the three-layer model and all the citations, lives in [`docs/spikes/2026-05-30-ai-cost-governance.md`](../spikes/2026-05-30-ai-cost-governance.md).*
