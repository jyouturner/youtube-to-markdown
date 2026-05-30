# Spike: Cost Governance for API-Based AI Applications

- **Date:** 2026-05-30
- **Status:** Research spike (no production code; starting point for further investigation + decisions)
- **Context:** Triggered by cost work on yt2md. We corrected a stale pricing table, found and fixed a prompt-cache that was billing without ever being read, wired the app to the Anthropic Admin Cost/Usage API, scoped it to a dedicated workspace, and built a client-side budget gate. This doc zooms out from those tactical fixes to the whole governance landscape.
- **Scope:** Cost control / financial governance of LLM-API-based apps. Provider examples are Anthropic-centric (that's what we use), but the architecture is provider-agnostic — the gateway layer in particular is explicitly multi-provider (Anthropic, OpenAI, others).

---

## 0. TL;DR — the findings that matter

1. **Cost control is a three-layer problem, not two.** "In-process vs out-of-process" is the right instinct, but the out-of-process half splits into two layers separated by one hard capability line: **can it reject a single request, in real time, on a budget basis?**
   - **Layer 1 — In-app controls:** fast, free, single-app. Soft.
   - **Layer 2 — A gateway you run** (between your app and the LLM API): the **only** layer that can hard-reject an individual request in real time. Provider-agnostic. This is the layer we hadn't named.
   - **Layer 3 — Provider-native** (Anthropic Admin API, workspaces, spend caps): authoritative billing + org guardrails, but **reporting-only, ~5-minute latency** — it cannot reject a request in real time.

2. **The Anthropic Admin/Cost API is monitoring, not enforcement.** Data lags ~5 min. The closest thing to a programmatic kill switch is **disabling an API key** (`POST /v1/organizations/api_keys/{id}` `status:"inactive"`), which stops the *next* request — not an in-flight stream. Setting spend/rate limits and creating keys are **Console-only** (no API).

3. **The real-time kill switch lives in a gateway you control.** A self-hosted proxy (e.g. **LiteLLM**, OSS) tracks spend per virtual key/team/user and returns an error instead of forwarding once a budget is exceeded. This is the single biggest gap-filler for "stop a runaway process."

4. **No public OAuth for the API; don't scrape the Console.** There is no user-sign-in token for the Messages/Admin API. Console-internal endpoints power the write ops the public API withholds, but automating them via a scraped session token is ToS-risky and brittle. Not a foundation for governance.

5. **Two unused in-process levers are high-value for our workload:** the **Batch API** (flat 50% off, ideal for non-interactive pipelines) and **prompt caching of the transcript** across pipeline stages. Plus `count_tokens` (free pre-flight pricing) and the `effort` dial.

---

## 1. The three-layer model

```
   ┌─────────────────┐     ┌──────────────────────┐     ┌───────────────────────┐
   │  Layer 1         │     │  Layer 2             │     │  Layer 3              │
   │  IN-APP          │ ──► │  GATEWAY (you run)   │ ──► │  PROVIDER-NATIVE      │
   │                  │     │                      │     │  (Anthropic / OpenAI) │
   │ token caps,      │     │ per-request $ cutoff │     │ workspaces,           │
   │ budget gate,     │     │ virtual-key budgets, │     │ spend caps (Console), │
   │ count_tokens,    │     │ rate limit, cache,   │     │ Admin/Cost API        │
   │ model routing    │     │ fallback, 1 log      │     │ (reporting, ~5m lag)  │
   └─────────────────┘     └──────────────────────┘     └───────────────────────┘
   real-time, single-app    real-time, ALL apps,         authoritative billing,
   soft enforcement         hard enforcement,            org guardrail,
                            provider-agnostic            NOT real-time
```

The capability line is **real-time per-request rejection**. Only Layer 2 has it for *all* traffic. Layer 1 has it for one app's own code. Layer 3 never does (it's billing/reporting + coarse account caps).

---

## 2. Layer 1 — In-process controls

Levers the app controls inside the request/response loop. Our current budget gate + usage logging live here.

| Lever | Mechanism | Cost impact | Caveat | Status for us |
|---|---|---|---|---|
| **`/v1/messages/count_tokens`** | Free endpoint; returns exact input-token count for a payload (incl. tools/images/system) | Price/route/trim *before* paying | Estimate only; **does not model output tokens** (the expensive side); separate rate limit | Unused — would let the gate price a video up front |
| **Batch API** | Async submit/poll/retrieve | **Flat 50% off** input+output; stacks with caching (~95%) | ≤24h turnaround; no streaming; `max_tokens≥1` | Unused — **ideal** for our non-interactive digest/panel/takeaway pipeline |
| **Prompt caching** | `cache_control` breakpoints | Read = **0.1×**; write = 1.25× (5m) / 2× (1h) | **Write-without-read = +25% net loss** (the bug we just fixed); min block 1,024–4,096 tok | Just removed a misuse; a *shared* transcript cache across stages would be a real win |
| **`effort` (low/med/high/xhigh/max)** | Soft dial over text + tool + thinking tokens | Biggest *in-flight* spend dial | Default `high`; thinking bills at **output** rates; on Opus 4.7/4.8 manual `budget_tokens` 400s — use `thinking:{type:"adaptive"}` | Reserve high for the panel, low elsewhere |
| **`max_tokens` / `stop_sequences` / structured outputs** | Hard output ceiling / terminator / grammar-constrained output | Bound worst-case output; kill malformed-JSON retries | Structured outputs are beta; `max_tokens` truncates mid-response | Partially used |
| **Streaming + client abort** | `stream:true`; cancel mid-flight | Stops a runaway generation early | **Billed for tokens already generated** before disconnect | — |
| **Timeouts & SDK retries** | Request timeout + auto-retry (default ~2) | Bounds hung calls | Retries multiply cost; a progressing call can be billed on each disconnect | Tighten on long calls |
| **Context editing / compaction** | Server-side `clear_tool_uses` / `clear_thinking`; client compaction | Caps growing input in long agent loops | Clearing invalidates cache prefix — use `clear_at_least` | N/A (no long loops) |
| **Model routing** | Haiku $1/$5 · Sonnet $3/$15 · Opus $5/$25 per MTok | **Single biggest lever** — 5× spread | ⚠️ Opus 4.7+ tokenizer emits **up to 35% more tokens** → effective cost > rate card | Known; verify Opus with `count_tokens` |
| **`inference_geo:"us"`** | Data-residency flag | **1.1× multiplier** on 4.6+ | Pure cost *adder* — only if residency required | Avoid unless needed |

**Blind spot of this layer (important):** our gate checks **only at digest start** and reads billing data that **lags ~5 min**, so it cannot stop a runaway *mid-pipeline*. That's not patchable in-process — it's the inherent ceiling of Layer 1. Closing it requires Layer 2.

---

## 3. Layer 2 — The gateway (the missing middle; the real kill switch)

A gateway/proxy sits inline on **every** LLM call, tracks cumulative spend per virtual key / team / user in its own datastore, and can **return an error instead of forwarding** once a budget is exceeded. That is the real-time hard cutoff neither Layer 1 (single-app only) nor Layer 3 (reporting-only) provides.

It is also the natural **provider-abstraction** point: one OpenAI-compatible endpoint in front of Anthropic *and* OpenAI *and* others, so governance, logging, caching, fallback, and chargeback are uniform regardless of backend. This is why it's the right forward investment if the app ever spans providers.

### Real-time hard-cutoff capability

| Gateway | Per-key/team/user budgets | Hard $ cutoff (rejects request)? | Self-host / free? | Notes |
|---|---|---|---|---|
| **LiteLLM (proxy)** | Yes — personal/team/member/customer/agent; multiple concurrent windows | **Yes** — exceeding `max_budget` blocks with an error; soft budgets alert only | **Yes (OSS, MIT)** | Canonical answer. Known edge: stale-spend / budget-bypass bugs when model names don't follow `provider/model` form (#27735, #24770) |
| **TrueFoundry** | Per team/user/model/app/env | **Yes** — throttle / downgrade-to-cheaper-model / **block** | Yes (self-host first) | Most flexible breach behavior; GitOps config |
| **Kong AI Gateway** | Consumer/group/model/provider | **Yes** — cost-based + token-based limiting from real response tokens | Yes (self-host; Enterprise feature) | Plugin-based; semantic cache; prompt compression |
| **Portkey** | USD or token budget per virtual key | **Yes** — key auto-expires at budget (**Enterprise tier only**) | SaaS + self-host (enterprise) | Governance gated behind enterprise pricing |
| **LangDB** | Per-workspace budgets on unified credits | Yes | SaaS + self-host (OSS, Rust) | Cost-optimized routing |
| **OpenRouter** | Per-key spend limits + prepaid credit balance | **Yes, structurally** — credits exhausted → requests fail | SaaS only (+5.5% fee) | Strongest multi-provider fallback |
| **Cloudflare AI Gateway** | Analytics by provider; no per-key $ budget | **No $ cutoff** — rate limit + cache + visibility only | SaaS (edge, generous free tier) | Easiest drop-in; not a dollar kill switch |
| **Helicone** | Cost analytics + rate limits per user/project | Rate limits yes | Yes (OSS) | ⚠️ **Maintenance mode** since Mar 2026 Mintlify acquisition — don't pick as a forward bet |

**For a personal/Anthropic-centric setup:** the lightest *real* kill switch is **self-hosted LiteLLM** in front of Anthropic with a per-key `max_budget` + a daily window. For **multi-provider**, the poles are OpenRouter (SaaS, prepaid hard cap) and LiteLLM (self-host).

**Gateway pros:** one policy surface across all apps + providers; real-time rejection; centralized log/cache/fallback; team/user attribution for chargeback; vendor portability.
**Gateway cons:** it becomes a critical-path dependency (latency + SPOF → needs HA); operational burden (run/patch/scale); spend-accuracy edge cases; another credential surface to secure.

---

## 4. Layer 3 — Provider-native (Anthropic Admin API): what it can and cannot do

Authoritative billing + org guardrails, but **reporting-only with ~5-min latency. It cannot reject a request in real time.** Internalizing this is the point of the spike — it bounds what the API we already adopted can do.

### The "kill a runaway" options

| Mechanism | Via API? | Stops in-flight request? | Granularity / latency |
|---|---|---|---|
| **Disable a key** (`POST /v1/organizations/api_keys/{id}` `status:"inactive"` or `"archived"`) | **Yes (Admin)** | ❌ No — next request fails 401 | One key; near-immediate for *new* requests; reversible. **The programmatic kill switch** |
| **Archive a workspace** (`/workspaces/{id}/archive`) | **Yes (Admin)** | ❌ next request only ("immediately revokes all keys") | Whole project; irreversible |
| **Set workspace rate limit** | ❌ **Console-only** | Throttles, doesn't halt | Workspace; readable via API, not settable |
| **Org spend cap** (customer-set or tier ceiling) | ❌ **Console-only** | Halts new requests at cap (429) | Whole org; **monthly** granularity |
| **Remove user** | Yes (Admin) | ❌ next request only; affects only Claude-Code per-user keys | Standard workspace keys persist when a user is removed |

**In-flight vs next request:** every control is an auth check evaluated *per request* — so disabling a key/workspace fails the *next* call; **no documented mechanism tears down an already-streaming generation.** For a single hung stream, only client-side abort works (Layer 1). Anthropic publishes **no SLA** for disable-propagation latency or in-flight termination (flagged by research as the one unpinnable fact).

### What's API-readable vs Console-only

- **API can:** list/get/update members, invites, workspaces (CRUD + archive), workspace members, **API keys (list/get/update-status)**; read usage + cost reports; read rate limits; per-user Claude-Code cost via the Analytics API.
- **API cannot (Console-only):** **create** API keys, **set** spend limits (org or workspace), **set** rate limits.

### Key endpoints (Admin key `sk-ant-admin…`, org-only)

| Purpose | Endpoint |
|---|---|
| Usage report | `GET /v1/organizations/usage_report/messages` (1m/1h/1d; group/filter by model, workspace, api_key, service_tier, context_window, inference_geo, speed) |
| Cost report | `GET /v1/organizations/cost_report` (1d only; USD in cents; group by workspace_id / description) |
| API keys | `GET/POST /v1/organizations/api_keys[/{id}]` (status: active / inactive / archived) |
| Workspaces | `…/workspaces` (+ `/{id}/archive`) |
| Rate limits (read) | `GET /v1/organizations/rate_limits`, `…/workspaces/{id}/rate_limits` |
| Claude Code per-user cost | `GET /v1/organizations/usage_report/claude_code` (daily, ~1h lag, per-user `estimated_cost`) |

---

## 5. Monitoring & observability tier (cannot enforce; valuable anyway)

Sits *beside* the traffic (pulls billing/usage APIs or ingests traces). **None can reject a request** — they answer "where did the money go / who owes it" and "what happened in this call."

- **FinOps-for-AI:** Anthropic's *named* Cost-API partners are **CloudZero, Datadog, Grafana Cloud, Honeycomb, Vantage**. They normalize Claude spend alongside cloud/other-LLM spend for allocation, forecasting, showback/chargeback. (Grafana Cloud's Anthropic integration is a fast, free way to get dashboards off the Cost API.)
- **LLM observability:** Langfuse (OSS), Datadog LLM Observability, Arize Phoenix (OSS), Braintrust, Helicone (now maintenance-mode). Capture per-trace token counts + *estimated* cost.
- **The strategic standard — OpenTelemetry GenAI semantic conventions:** standardized `gen_ai.usage.input_tokens` / `output_tokens` attributes + `gen_ai.client.token.usage` metric. Status: **experimental; no standard `cost` attribute yet** — cost is computed downstream from standardized tokens × a pricing table (exactly what our calibrated table does). **Why it matters:** instrument once with `gen_ai.*` → any OTel backend (Langfuse, Phoenix, Honeycomb, Datadog, Grafana) computes cost identically, no re-instrumentation, no lock-in. This is the portable foundation to adopt early.

---

## 6. The OAuth / Console-internal-API question (honest assessment)

You hypothesized that since the Console is API-powered, a user could sign in, get a token, and call the same APIs. Findings:

- **No public OAuth for the Messages/Admin API.** The Admin key is the only supported programmatic-admin credential.
- Claude Code *does* mint OAuth tokens (`sk-ant-oat01-`, PKCE flow), but **the public API rejects them** ("OAuth authentication not supported") — they work only inside Claude Code / Agent SDK subscription flows.
- The Console clearly calls **internal/undocumented endpoints** for the write ops the public API withholds (set spend/rate limits, create keys).
- **Scraping a browser session token to drive those: not recommended.** ToS-gray-to-red (accessing interfaces deliberately not exposed); brittle (paths/CSRF/bot-protection change without notice; session tokens die on re-auth/MFA); a credential-handling liability likely to trip anomaly detection.
- The sanctioned pattern for federated short-lived creds is **Workload Identity Federation** — but it's *tunnel-scoped* (`org:manage_tunnels`), not a general Admin replacement.
- **Verdict:** for programmatic spend/rate-limit *enforcement*, it isn't available today. File a feature request rather than build on internal endpoints.

---

## 7. What we built this session (grounding)

| Artifact | Layer | Note |
|---|---|---|
| Corrected `DEFAULT_MODEL_PRICING` | 1 | Opus 4.7/4.8 were priced at the old $15/$75 tier; corrected to $5/$25 (3× overstatement in the log) |
| Removed never-read prompt cache on digest/panel/takeaway | 1 | Was paying 1.25× cache-write for 0 reads; ~$0.033/video |
| Self-calibrating pricing (`refresh-pricing` + `pricing_cache.json`) | 1 + 3 | Derives real $/Mtok from the Cost API so the table can't silently go stale |
| Client-side budget gate (warn $15 / block $18) | 1 (+3 data) | Reads workspace month-to-date from the Cost API; soft, start-of-digest only |
| Dedicated `yt2md` workspace + scoped key + $20 Console cap | 3 | Per-app cost attribution + an org-level 429 backstop |

These cover **Layer 1 + Layer-3 reporting**. The gap — **real-time enforcement (Layer 2)** — is unfilled by design; for a single-user tool the Console $20 cap is an adequate backstop.

---

## 8. Decision framework — when each layer is worth it

- **Always:** Layer 1 in-app discipline (model routing, output bounds, caching-done-right, a soft budget gate) + Layer 3 for authoritative billing reconciliation and an org spend-cap backstop. Low cost, high leverage.
- **Add Layer 2 (gateway) when any of:** >1 app or >1 user shares a key; you need a *hard* real-time per-key/team budget; you span multiple providers (Anthropic + OpenAI) and want uniform governance/fallback/logging; you need chargeback. Below that bar, a gateway's SPOF/HA/ops cost outweighs the benefit.
- **For yt2md specifically (personal, Anthropic-only):** stay at Layer 1 + Layer 3. Note the gateway as the documented next step if scope grows.

---

## 9. Open questions / next investigations

1. **Gateway deep-dive (top priority per project direction):** stand up a local **LiteLLM** proxy in front of Anthropic; validate per-key `max_budget` hard rejection, daily+monthly windows, OpenAI-compatible passthrough, and the OTel export. Verify the stale-spend bugs (#27735/#24770) don't affect a single-key setup. Decide build-vs-skip for our scale.
2. **Undocumented Anthropic behavior:** measure key-disable propagation latency and whether an in-flight stream is torn down on disable (no published SLA).
3. **OTel GenAI conventions:** track the timeline for a standardized `cost` attribute; decide whether to emit `gen_ai.*` from yt2md now for portability.
4. **In-process wins to revisit if LLM work returns app-side:** Batch API (50% off, fits the pipeline) and shared transcript prompt-caching across stages.
5. **Multi-provider question:** if the app ever adds OpenAI, the gateway becomes the abstraction + governance point — re-evaluate Layer 2 at that moment.

---

## 10. Sources

**Anthropic (official):**
- Admin API overview — https://platform.claude.com/docs/en/build-with-claude/administration-api
- Usage & Cost API — https://platform.claude.com/docs/en/build-with-claude/usage-cost-api
- Update API Key — https://platform.claude.com/docs/en/api/admin-api/apikeys/update-api-key
- Rate Limits API — https://platform.claude.com/docs/en/manage-claude/rate-limits-api
- Workspaces (archive, spend limits) — https://platform.claude.com/docs/en/manage-claude/workspaces
- Claude Code Analytics API — https://platform.claude.com/docs/en/manage-claude/claude-code-analytics-api
- Pricing — https://platform.claude.com/docs/en/about-claude/pricing
- Token counting — https://platform.claude.com/docs/en/build-with-claude/token-counting
- Effort — https://platform.claude.com/docs/en/build-with-claude/effort
- Adaptive thinking — https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking
- Structured outputs — https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- Streaming — https://platform.claude.com/docs/en/build-with-claude/streaming
- Service tiers — https://platform.claude.com/docs/en/api/service-tiers
- Prompt caching — https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Batch processing — https://platform.claude.com/docs/en/build-with-claude/batch-processing
- Context editing — https://platform.claude.com/docs/en/build-with-claude/context-editing
- Billing FAQ (disconnect billing) — https://support.claude.com/en/articles/8114526-how-will-i-be-billed-for-claude-api-use

**Gateways:**
- LiteLLM — https://docs.litellm.ai/docs/proxy/users · https://docs.litellm.ai/docs/proxy/virtual_keys · https://docs.litellm.ai/docs/proxy/cost_tracking · bugs https://github.com/BerriAI/litellm/issues/27735 , https://github.com/BerriAI/litellm/issues/24770
- Portkey — https://portkey.ai/docs/product/ai-gateway/virtual-keys/budget-limits
- Cloudflare AI Gateway — https://developers.cloudflare.com/ai-gateway/features/ · https://developers.cloudflare.com/ai-gateway/features/rate-limiting/
- Kong AI Gateway — https://developer.konghq.com/plugins/ai-rate-limiting-advanced/ · https://konghq.com/blog/engineering/token-rate-limiting-and-tiered-access-for-ai-usage
- OpenRouter — https://openrouter.ai/docs/api/reference/limits · https://openrouter.ai/docs/faq
- TrueFoundry — https://www.truefoundry.com/docs/ai-gateway/budgetlimiting
- LangDB — https://langdb.ai/why-ai-gateway/
- Helicone — https://docs.helicone.ai/guides/cookbooks/cost-tracking · https://github.com/Helicone/helicone

**FinOps / observability:**
- CloudZero × Anthropic — https://www.cloudzero.com/blog/cloudzero-anthropic/
- Datadog — https://www.datadoghq.com/blog/anthropic-usage-and-costs/
- Grafana Cloud — https://grafana.com/blog/how-to-monitor-claude-usage-and-costs-introducing-the-anthropic-integration-for-grafana-cloud/
- Vantage — https://docs.vantage.sh/connecting_anthropic
- Langfuse — https://langfuse.com/docs/observability/features/token-and-cost-tracking
- OpenTelemetry GenAI — https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/ · https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/ · https://opentelemetry.io/blog/2026/genai-observability/

**Third-party pricing analyses (corroborating, verify against own measurements):**
- Finout (Opus 4.7 tokenizer) — https://www.finout.io/blog/claude-opus-4.7-pricing-the-real-cost-story-behind-the-unchanged-price-tag
- CloudZero (Opus 4.7 pricing) — https://www.cloudzero.com/blog/claude-opus-4-7-pricing/

---

*Spike compiled from three parallel research threads (Anthropic governance surface · in-process levers · external ecosystem). Endpoint mechanics, parameters, multipliers, and billing rules are from official Anthropic docs; pricing percentages from corroborating third-party analyses (verify with `count_tokens`). One fact remains unpinnable: Anthropic publishes no SLA for API-key-disable propagation latency or in-flight stream termination.*
