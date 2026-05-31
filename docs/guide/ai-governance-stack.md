# The Centralized AI Application Governance Stack — What It Covers & Build-vs-Buy

*Last updated: 2026-05-30*

**Who this is for:** platform, security, and FinOps leaders deciding how to govern AI applications across an organization — and engineers who want to understand where a gateway like LiteLLM fits in the larger picture.

**Companion to** [`anthropic-cost-control.md`](anthropic-cost-control.md): that guide is the *cost-control* layer for one app on one provider; this one zooms out to the *whole governance system* across many apps, teams, and providers.

> **Sourcing caveat:** the vendor landscape moves fast and several of the richest pricing/TCO comparisons cited here come from vendor blogs (notably TrueFoundry comparing itself to competitors, and Portkey). Treat exact dollar figures and competitive framings as **directional, not audited** — verify against your own RFP. Structural facts (OSS license, self-host model, acquisitions, framework deadlines) are cross-checked against primary or multiple sources.

---

## 1. The framing: two planes, one connective tissue

Governance discussions get muddled because they conflate two things that live in different places and have different latency, ownership, and tooling:

- **Runtime enforcement plane** — inline, in the request path, low-latency. Inspects, blocks, redacts, routes, and budgets every call. This is where a runaway is *stopped* and a jailbreak is *caught*.
- **Policy & compliance plane** — out-of-band, the system of record. Model inventory, risk classification, audit trails, approvals, model cards, framework mappings. This is where you *prove* governance to an auditor or regulator.

**The AI gateway is the connective tissue.** It enforces one policy set at runtime regardless of model *and* emits the audit trail the compliance plane consumes. The 2026 consensus: once an org runs dozens of LLM-backed services, governance "cannot be retrofitted at the application layer — it has to live at the gateway." Hold this distinction; most buying mistakes come from expecting a documentation-plane product (Credo AI, OneTrust) to enforce at runtime, or a gateway to satisfy a compliance auditor on its own.

---

## 2. Reference architecture — what the stack covers

```
                    ┌─────────────────────────────────────────────────┐
  IDENTITY / ACCESS │ SSO/SCIM · RBAC · virtual keys · secrets vault     │  cross-cutting
                    └─────────────────────────────────────────────────┘
   ┌──────────────────────────── CONTROL PLANE ─────────────────────────────┐
   │  AI GATEWAY — routing/fallback · virtual keys · per-team budgets +       │
   │  hard cutoffs · rate limits · semantic/response caching · provider        │
   │  abstraction · unified audit logging                                     │
   └──────────────────────────────────────────────────────────────────────────┘
         │                  │                   │                   │
  ┌────────────┐    ┌───────────────┐   ┌───────────────┐   ┌────────────────┐
  │  QUALITY   │    │ SAFETY / SEC  │   │ COST / FINOPS │   │  AGENT / MCP   │
  │ observ. +  │    │ guardrails +  │   │ allocation,   │   │  GOVERNANCE    │
  │ evals +    │    │ AI security   │   │ chargeback,   │   │  registries,   │
  │ prompt mgmt│    │ (PII,inject,  │   │ unit econ,    │   │  tool RBAC,    │
  │            │    │  moderation)  │   │ forecasting   │   │  approvals     │
  └────────────┘    └───────────────┘   └───────────────┘   └────────────────┘
   ┌──────────────────────── POLICY / COMPLIANCE PLANE ─────────────────────┐
   │ model inventory · risk tiering · audit logs · approvals · model cards   │
   │ EU AI Act · NIST AI RMF · ISO/IEC 42001 · SOC 2   (documentation + GRC) │
   └──────────────────────────────────────────────────────────────────────────┘
     operated by: AI Platform Team + AI CoE + FinOps practice + Governance board
     exposed via: internal developer portal (self-service governed key issuance)
```

Each domain is a real sub-market with its own options:

| Domain | What it governs | The non-obvious bit |
|---|---|---|
| **Gateway (control plane)** | the chokepoint: budgets, access, routing, caching, audit | Gartner projects **70% of multi-model teams will use an AI gateway by 2028** (≈25% in 2025); 2025 TAM only ~$50–100M — early, fast-growing |
| **Observability + evals + prompt mgmt** | tracing (audit evidence), regression gates (quality SLAs), versioned prompts (the approvable change unit) | "Prompt versioning without evals is just diff tracking" — governance needs all three *linked* |
| **Safety / AI security** | PII redaction, prompt-injection/jailbreak defense, moderation, grounding, DLP | Map each control to an **OWASP LLM Top-10 (2025)** code so coverage is demonstrable to auditors |
| **FinOps for AI** | allocation, showback/chargeback, unit economics, forecasting, anomaly detection, commitments | A **formal FinOps Foundation Scope (2025)**; AI spend actively managed by **63% of practitioners, up from 31%** |
| **Agent / MCP governance** | tool/agent registries, per-tool RBAC, discovery, approval, invocation audit | **The 2026 frontier** — AWS Agent Registry, Databricks Unity, Azure API Center, agentgateway all ship this |
| **Identity / access** | SSO/SCIM, RBAC, virtual keys as tenant-isolation primitive, secrets in a vault | Issue **scoped virtual keys**, never raw provider keys |
| **Policy / compliance** | inventory, risk tiering, audit, approvals, framework mapping | **EU AI Act high-risk obligations apply 2 Aug 2026** — the hard deadline anchoring requirements |
| **Operating model** | ownership: platform team + CoE + FinOps + governance board; self-service portal | **Hub-and-spoke** (central standards, BU autonomy) is the recommended shape |

---

## 3. Options per layer — build (OSS) vs buy (commercial) vs cloud-native

### 3.1 Gateway / control plane

| Option | Type | Governance depth |
|---|---|---|
| **LiteLLM** (proxy) | OSS (MIT core; SSO/SAML/SCIM, audit-log retention, scoped guardrails are Enterprise) | **Strongest OSS governance** — free virtual keys, budgets, rate limits, spend logging |
| **agentgateway** (Linux Foundation) | OSS (Apache 2.0) | OSS *agent*-governance frontier: LLM+MCP+A2A, per-team budgets, OIDC, OPA, tamper-evident audit; young |
| Kong AI Gateway / Envoy AI Gateway / Higress | OSS core | Routing-strong; AI governance (semantic cache, prompt guard, analytics) **paywalled or plugin-gated** |
| **Portkey** | Commercial SaaS | High at **Enterprise ($2K–$10K+/mo)**; RBAC/SSO/VPC/SOC2 are Enterprise-only; critiqued as app-scoped, weak cross-team attribution |
| **TrueFoundry** | Commercial (self-host/VPC/on-prem) | Strong org-wide: LLM+MCP+agent gateway, team budgets, SOC2/HIPAA/EU-AI-Act; quote-based |
| Cloudflare AI Gateway | Commercial (edge) | Routing/observability + rate-limit + cache; **lighter on org-wide budgets/RBAC** — not a $ kill switch |
| OpenRouter | Commercial (marketplace) | Routing/fallback + per-key spend caps via prepaid credits; **not enterprise governance** |
| **AWS Bedrock AgentCore Gateway** + Agent Registry | Cloud-native | Managed MCP tool server (OAuth/IAM), semantic tool discovery, CloudTrail audit; **Agent Registry** = governed agent/tool catalog + approvals |
| **Azure API Management AI Gateway** (in Foundry) | Cloud-native | Explicit `llm-token-limit` (TPM + token quota per key), semantic cache, load-balance + circuit-breaker, Content Safety, token-metric logging |
| **GCP Apigee AI** + Vertex | Cloud-native | Token quotas, semantic caching, **Model Armor** (OWASP-LLM sanitization), MCP bridge |
| **Databricks Unity / Mosaic AI Gateway** | Cloud-native | Central governance of LLM endpoints + agents + MCP in Unity Catalog; guardrails GA; high lock-in |

*Cloud-native gateways give the deepest native governance if you're already in that cloud (IdP, content-safety, audit integrated), at the cost of lock-in. Azure APIM has the most explicit per-consumer token-budget policy; AWS/Databricks lead on agent/MCP estate governance.*

### 3.2 Observability + evaluation + prompt management

| Option | Type | Role |
|---|---|---|
| **Langfuse** | OSS (MIT) + cloud | The OSS spine — tracing + evals + prompt registry; full self-host (SOC2/ISO only on cloud) |
| **OpenLLMetry / Traceloop** | OSS (Apache 2.0) | Purest **OTel-native** instrumentation — export to any backend, no lock-in. Plumbing, not a product |
| Arize Phoenix / AX · DeepEval · Ragas | OSS | Phoenix (span tracing + drift), DeepEval (CI eval gates), Ragas (RAG metrics) |
| **Braintrust** | Commercial (hybrid VPC) | Unifies trace + eval + prompt gates; SOC2 II + HIPAA — strongest single-platform enterprise story |
| Datadog LLM Observability · LangSmith · Galileo · Patronus | Commercial | Datadog for Datadog shops; Galileo/Patronus for low-latency inline + online evals |
| PromptLayer · Agenta · Portkey | OSS/SaaS | Prompt versioning/registry; Agenta has VCS-style branching |

> ⚑ **Two casualties to route around:** **Helicone** went maintenance-mode (Mintlify acquisition, Mar 2026) — OSS/self-host live, but no roadmap. **Humanloop** was acqui-hired by Anthropic and **deleted** (Sept 2025; its capabilities live on as the Anthropic Console's Workbench/Evaluations tabs). Don't pick either as a forward bet.

> ⚑ **The standard — OpenTelemetry GenAI conventions — is still experimental (2026) and has NO standardized `cost` attribute.** `gen_ai.usage.input_tokens`/`output_tokens` and `gen_ai.client.token.usage` exist; cost is still computed downstream from tokens × a price table (exactly the calibrated table from the cost-control guide). For the **audit-trail layer specifically**, favor OSS/self-host or OTel-native exports — compliance records must outlive the vendor (cf. Helicone, Humanloop within ~12 months).

### 3.3 Safety / AI security

| Option | Type | Catches |
|---|---|---|
| **NVIDIA NeMo Guardrails** · **Guardrails AI** · **LLM Guard** · **Presidio** (PII) | OSS | Programmable rails / output validators / input+output scanners / PII detect+redact |
| **Lakera Guard** → Check Point · **Prompt Security** → SentinelOne · **Protect AI** → Palo Alto · **Robust Intelligence** → Cisco AI Defense | Commercial (all **acquired by security incumbents in 2025**) | Injection/jailbreak/PII/DLP at runtime |
| **AWS Bedrock Guardrails** · **Azure AI Content Safety + Prompt Shields** · **GCP Model Armor** | Cloud-native | Content filters, denied topics, PII, contextual-grounding/hallucination checks |
| **PyRIT** (Microsoft) · **Garak** (NVIDIA) · Promptfoo | OSS red-team | Out-of-band, pre-deployment attack testing (map findings to OWASP LLM Top-10) |

> ⚑ The standalone runtime-guardrail market **consolidated into security suites** in 2025 (all four marquee independents acquired). **Buy guardrail *detection*** — it's a moving research target; building injection/PII classifiers in-house rarely keeps pace.

### 3.4 FinOps for AI

| Option | Type | Strength |
|---|---|---|
| Native cost APIs + cloud budgets | Build | Table-stakes, free — but stops at billing **aggregates** |
| **CloudZero** | Commercial | **Strongest unit-economics** (cost per customer/feature/request) — needs instrumentation |
| Vantage · Finout · Datadog CCM | Commercial | Allocation/showback/chargeback, virtual tagging; Datadog observability-first (weaker allocation) |

*Two tool families, most orgs need one of each: finance-facing FinOps platforms (allocation/chargeback) and engineer-facing observability (per-request tracing). Unit economics — cost-per-customer/feature — is the underserved frontier.*

### 3.5 Policy / compliance (the documentation plane)

| Option | Plane | Bundles |
|---|---|---|
| **Credo AI** | Policy | Policy-to-control orchestration, Policy Packs (EU AI Act / ISO 42001 / NIST); OEM'd into IBM watsonx.governance |
| **OneTrust AI Governance** · **IBM watsonx.governance** · **Holistic AI** | Policy/GRC | Inventory, risk assessment, audit, bias detection, agent monitoring |
| **Dynamo AI** · **Knostic** · **Credal** | **Runtime + policy** | The few "all-in-one" platforms that also enforce inline (guardrails, access-aware retrieval, oversharing prevention) |

> ⚑ Most "all-in-one AI governance" platforms (Credo AI, OneTrust, IBM, Holistic) are **documentation plane only** — they do *not* sit inline at high throughput. A complete stack needs **both** a governance/GRC platform *and* a gateway+guardrail runtime layer.

### 3.6 Compliance frameworks (what drives the requirements)

| Framework | Nature | Translates into |
|---|---|---|
| **EU AI Act** | Binding law; **high-risk obligations 2 Aug 2026** | risk management, data governance, **logging**, human oversight, conformity assessment, registration. Fines up to €35M / 7% turnover |
| **NIST AI RMF** | Voluntary (Govern/Map/Measure/Manage) | AI inventory, model cards, per-system risk assessments, audit logs |
| **ISO/IEC 42001** | Certifiable AI management system | org-wide AIMS processes (ISO 42006:2025 qualifies the auditors) |
| **SOC 2** | Attestation | logical access controls on the prompt path, training-data access controls; Confidentiality + Privacy criteria |

*Build one control set and map it to all three — NIST publishes an official AI-RMF→ISO-42001 crosswalk, and both are the common implementation route to EU AI Act compliance.*

---

## 4. Build vs buy — the economics

**The load-bearing cost is headcount, not license.** A representative DIY stack (LiteLLM + Langfuse + Guardrails AI + OTel + an internal portal) for a ~200-person org, ~500M tokens/mo, runs roughly **$150–200K in year 1 / ~$500K over 3 years** — and **~$60–125K of that is the 0.25–0.5 FTE** to operate a *stateful, HA, critical-path* service (Postgres + Redis + multi-region + on-call + upgrade discipline). License and infra are the smaller lines. *(Figures are vendor-modeled; directionally corroborated across sources.)*

**The crossover (cross-source consensus):**

```
< ~1M req/mo, no platform team      → BUY managed. DevOps time > license savings.
Already run K8s + IaC + platform     → BUILD on OSS. Marginal ops cost is low.
> ~50M req/mo, cost-sensitive        → BUILD/self-host. Managed metering gets punishing.
Heavy compliance + small team        → BUY enterprise (offload certs/SLAs/audit infra).
All-in on one cloud / Databricks     → BUY cloud-native (accept lock-in for integration speed).
Multi-cloud, want portability        → BUY a portable platform (lower lock-in than cloud-native).
```

| Factor | Favors BUILD (OSS) | Favors BUY |
|---|---|---|
| Scale | Very high (>50M/mo) | Low–mid |
| Platform team | Exists, mature | None / thin |
| Compliance | Can self-certify | Need vendor SOC2/HIPAA/SLAs |
| Multi-cloud | Yes | Single cloud → cloud-native |
| Time-to-market | Flexible | Urgent |
| Lock-in tolerance | Low | Higher acceptable |

> ⚑ **Watch the metering geometry trap.** Portkey meters *logs* (agent workflows blow past caps → observability blind spots), Kong meters *requests* (one agent turn = 20+ internal calls), Databricks double-charges DBU + compute. The pricing unit can dominate the bill in agentic workloads.

---

## 5. Operating model — who owns it

The emerging consensus is **hub-and-spoke**: a lean central team sets standards (governance, evals, guardrails, FinOps); business units build use cases on the shared platform.

| Role | Owns |
|---|---|
| **AI Platform Team** | the internal AI gateway **as a product** — the control plane, SLAs, key issuance |
| **AI Center of Excellence (CoE)** | strategy, governance *standards*, best practices, enablement (executive-sponsored). *Warning: CoEs become bottlenecks when they own delivery instead of standards.* |
| **AI Governance Committee / Board** | evaluates initiatives on value/feasibility/risk; allocates capacity; resolves BU conflicts |
| **FinOps practice** | allocation, showback/chargeback, forecasting, commitment management |

**The portal pattern:** expose the gateway through an **internal developer portal** with **self-service governed key issuance** — developers browse a catalog, sandbox-test, generate scoped keys, and monitor their own usage without contacting the platform team. Backstage (~89% of the IDP market) is the de-facto base; in 2026 portals are adding MCP tool registration + OAuth M2M flows for agent consumption.

---

## 6. The maturity path (where most orgs actually are)

88% of orgs use AI somewhere, but only **~12% call their governance mature; ~81% sit in the earliest stages.** The evolution and the trigger for each jump:

| Stage | State | Symptoms | Trigger to advance |
|---|---|---|---|
| **0. Ungoverned sprawl** | per-team raw keys, no shared guardrails | overlapping copilots, **shadow/orphaned agents, permission creep**, surprise bills | **a cost shock or a security/compliance incident** |
| **1. Central gateway** | one egress: virtual keys, budgets, basic cost tracking | visibility appears; sprawl slows | need for chargeback / audit / multi-team allocation |
| **2. + Observability & FinOps** | tracing/evals + showback + anomaly detection | cost-per-feature visible; forecasting begins | regulatory pressure; agentic scale |
| **3. Full governance platform** | guardrails, RBAC, audit, MCP/agent governance, self-service portal | org-wide standards + self-service | steady state — continuous optimization |

**Trigger logic in one line:** *cost shock* drives 0→1; *finance/audit demand for chargeback* drives 1→2; *regulation + agentic complexity* drives 2→3. (A formal academic model — the arXiv "Agentic AI Governance Maturity Model," 5 levels × 12 domains, grounded in NIST AI RMF + ISO 42001 — reports Level 4–5 orgs show ~94% lower agent-sprawl and ~96% fewer risk incidents than Level 1.)

---

## 7. Where a single-app gateway (and this project) fits

To connect this to the cost-control work in this repo: **LiteLLM is the control-plane core of Stage 1**, plus a chunk of Stage 2's cost tracking — but a full governance system is the entire stack above it. What we built for this project (billing-calibrated pricing, a workspace-scoped budget gate, the validated LiteLLM kill switch) is essentially a **hand-built Stage 0→1 transition for one application**: the chokepoint, the cost meter, the spend guardrail. Scaling that to an organization is precisely the assemble-vs-buy decision in Part 4 — and the maturity ladder in Part 6 is the map for *when* each additional layer earns its keep.

**The one-sentence version:** a centralized AI governance system is a *gateway you enforce at* + *observability/evals you measure with* + *guardrails you're safe behind* + *FinOps you allocate with* + *a compliance plane you prove with* — operated by a platform team and exposed through a self-service portal — and you climb that stack one trigger (cost shock → audit demand → regulation) at a time, not all at once.

---

## Sources (selected; full citations in the research threads behind this doc)

**Category & gateways:** Gartner Market Guide for AI Gateways (Oct 2025); [LiteLLM](https://github.com/BerriAI/litellm) / [Enterprise](https://docs.litellm.ai/docs/enterprise); [agentgateway.dev](https://agentgateway.dev/); [Azure APIM AI gateway](https://learn.microsoft.com/en-us/azure/api-management/genai-gateway-capabilities); [AWS AgentCore Gateway](https://aws.amazon.com/blogs/machine-learning/introducing-amazon-bedrock-agentcore-gateway-transforming-enterprise-ai-agent-tool-development/) / [Agent Registry](https://aws.amazon.com/about-aws/whats-new/2026/04/aws-agent-registry-in-agentcore-preview/); [Apigee AI](https://cloud.google.com/solutions/apigee-ai); [Databricks Unity AI Gateway](https://www.databricks.com/product/artificial-intelligence/ai-gateway); [Portkey/TrueFoundry/Kong comparisons (vendor)](https://www.truefoundry.com/blog/a-definitive-guide-to-ai-gateways-in-2026-competitive-landscape-comparison).
**Observability/evals:** [Langfuse](https://langfuse.com/); [OpenLLMetry](https://github.com/traceloop/openllmetry); [OTel GenAI conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/); [Braintrust](https://www.braintrust.dev/); Helicone→Mintlify ([Mintlify](https://www.mintlify.com/blog/mintlify-acquires-helicone)); Humanloop shutdown ([TechCrunch](https://techcrunch.com/2025/08/13/anthropic-nabs-humanloop-team-as-competition-for-enterprise-ai-talent-heats-up/)).
**Safety/security:** [OWASP Top 10 for LLM Apps 2025](https://genai.owasp.org/llm-top-10/); [NeMo Guardrails](https://github.com/NVIDIA-NeMo/Guardrails); [Bedrock Guardrails](https://aws.amazon.com/bedrock/guardrails/); [Presidio](https://microsoft.github.io/presidio/); Cisco/Robust Intelligence, SentinelOne/Prompt Security, Palo Alto/Protect AI acquisitions (vendor/SEC primary).
**FinOps/compliance/operating model:** [FinOps for AI](https://www.finops.org/wg/finops-for-ai-overview/); [CloudZero FinOps for AI](https://www.cloudzero.com/blog/finops-for-ai/); [EU AI Act timeline](https://artificialintelligenceact.eu/implementation-timeline/); [NIST AI RMF↔ISO 42001 crosswalk](https://airc.nist.gov/docs/NIST_AI_RMF_to_ISO_IEC_42001_Crosswalk.pdf); [AI CoE (Microsoft CAF)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/ai/center-of-excellence); [AI governance maturity (arXiv)](https://arxiv.org/abs/2604.16338).

*Compiled from four parallel research threads (gateway/control-plane · observability/evals · safety/security/compliance · FinOps/platforms/operating-model), 2026. Vendor-authored comparisons are flagged inline; pricing is point-in-time. Companion: [`anthropic-cost-control.md`](anthropic-cost-control.md), [the cost-governance spike](../spikes/2026-05-30-ai-cost-governance.md), and [the blog narrative](../blog/2026-05-30-what-a-061-api-call-taught-me.md).*
