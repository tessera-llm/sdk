# Tessera SDK — drop-in LLM cost-optimization proxy

Tessera is an LLM inference API gateway that sits in your request path. It auto-routes to cheaper-equivalent models, caches repeated prompts, compresses context, and batches eligible calls. Every request is measured for cost delta against the model you originally asked for. **Pricing is a flat monthly subscription priced by your gross monthly token volume. You keep 100% of the measured savings — savings is your ROI proof, not our billing basis.**

[![PyPI](https://img.shields.io/pypi/v/tessera-llm-proxy.svg?label=PyPI)](https://pypi.org/project/tessera-llm-proxy/)
[![PyPI downloads](https://img.shields.io/pypi/dm/tessera-llm-proxy.svg?label=pip%20installs)](https://pypi.org/project/tessera-llm-proxy/)
[![Python](https://img.shields.io/pypi/pyversions/tessera-llm-proxy.svg)](https://pypi.org/project/tessera-llm-proxy/)
[![npm](https://img.shields.io/npm/v/@tessera-llm/tessera-sdk.svg?label=npm)](https://www.npmjs.com/package/@tessera-llm/tessera-sdk)
[![npm downloads](https://img.shields.io/npm/dm/@tessera-llm/tessera-sdk.svg?label=npm%20installs)](https://www.npmjs.com/package/@tessera-llm/tessera-sdk)
[![Node](https://img.shields.io/node/v/@tessera-llm/tessera-sdk.svg)](https://www.npmjs.com/package/@tessera-llm/tessera-sdk)
[![CI](https://github.com/tessera-llm/tessera-sdk/actions/workflows/ci.yml/badge.svg)](https://github.com/tessera-llm/tessera-sdk/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![GitHub stars](https://img.shields.io/github/stars/tessera-llm/tessera-sdk?style=flat&color=18181b&labelColor=fafafa)](https://github.com/tessera-llm/tessera-sdk)

**Free Sandbox tier: 60M tokens / month, no card required.** Get a key at [tesseraai.io/dev](https://tesseraai.io/dev).

---

## See it in action

[![Tessera launch demo — 41-second walkthrough](https://tesseraai.io/launch-reel-poster.jpg)](https://tesseraai.io/launch-reel.mp4)

▶ 41-second walkthrough: live counter ticks · baseline $74,800 → actual $30,000 ($44,800 saved, 60% reduction) · audit-immutable savings ledger. Click to play.

---

## 60-second runnable example

```python
# Python — pip install "tessera-llm-proxy>=0.1.0,<0.2"
import tessera, openai
tessera.activate("tk_your_tessera_key")        # one line

client = openai.OpenAI()                       # your existing code
client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
# Same response shape. Behind the scenes: route + cache + compress + measure.
# Savings ledger ticks live at ledger.tesseraai.io/portal/audit.
```

```ts
// Node / TypeScript — npm install @tessera-llm/tessera-sdk@^0.1.0
import { activate } from "@tessera-llm/tessera-sdk";
import OpenAI from "openai";

activate("tk_your_tessera_key");                // one line

const client = new OpenAI();                    // your existing code
await client.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: "Hello" }],
});
```

No SDK swap, no wrapper class, no decorator. The provider client you already use gets its `baseURL` + `X-Tessera-Key` header injected at construction time; everything else runs unchanged.

Architecture rationale + verification procedure (audit-immutable savings, multi-source pricing catalog, thin-SDK-by-design): see [ARCHITECTURE.md](./ARCHITECTURE.md).

---

<details>
<summary>Table of contents</summary>

- [Who this is for](#who-this-is-for)
- [Install](#install)
- [One-line integration](#one-line-integration)
- [30-second curl test](#30-second-curl-test)
- [Quality SLA — auto-rollback + 10% credit](#quality-sla--auto-rollback--10-credit)
- [Worked example — what a flat subscription looks like](#worked-example--what-a-flat-subscription-looks-like)
- [What Tessera does to each request](#what-tessera-does-to-each-request)
- [How it works (60 seconds)](#how-it-works-60-seconds)
- [Pricing](#pricing)
- [Supported providers](#supported-providers)
- [Compared to LLM observability tools](#compared-to-llm-observability-tools)
- [Frameworks & examples](#frameworks--examples)
- [Type safety](#type-safety)
- [Tessera in one paragraph (for search engines)](#tessera-in-one-paragraph-for-search-engines)
- [FAQ](#faq)
- [Documentation](#documentation)
- [Contributing](#contributing)
- [License](#license)
- [About Tessera](#about-tessera)

</details>

---

## Who this is for

- **AI-native SaaS** spending $5k+/month on OpenAI / Anthropic / Gemini and wanting that bill cut without re-architecting.
- **Vertical AI agents** (sales, support, voice, customer success) where margin compresses as call volume scales.
- **Engineering teams** who want an honest cost-reduction layer, not another observability dashboard.
- **Solo developers, side-project builders, and hobbyists** with personal Anthropic / OpenAI / Mistral API accounts — the 60M-tokens-per-month free tier covers most personal projects entirely. No card up front, no per-token fee, no future bill surprise.

**Not for:**
- **Consumer subscriptions** (Claude Pro, ChatGPT Plus, Gemini Advanced) — Tessera proxies API requests; subscriptions don't expose an API and aren't billed per token. Tessera cannot route subscription traffic.
- **Air-gapped on-prem deployments** — we're a hosted proxy only.

---

## Install

| Language | Install |
|---|---|
| Python | `pip install "tessera-llm-proxy>=0.1.0,<0.2"` |
| Node / TypeScript | `npm install @tessera-llm/tessera-sdk@^0.1.0` |

Pre-1.0 semver: minor releases may include breaking changes. Pin a floor + ceiling in production.

## One-line integration

### Python

```python
import tessera
tessera.activate("tk_your_tessera_key")

# Existing code runs unchanged. Any openai.OpenAI(), anthropic.Anthropic(),
# mistralai.Mistral(), groq.Groq(), cohere.Client() constructed AFTER this
# call routes through Tessera transparently. Your provider keys stay in
# the environment as usual.
```

### Node / TypeScript

```ts
import { activate } from "@tessera-llm/tessera-sdk";
activate("tk_your_tessera_key");

// Same shape: new OpenAI(), new Anthropic(), new Mistral(), etc. — all
// patched at load time. Bring-your-own provider keys.
```

---

## 30-second curl test

Get your key first at [tesseraai.io/dev](https://tesseraai.io/dev) — takes a minute, no card. Then:

```bash
curl https://api.tesseraai.io/v1/openai/chat/completions \
  -H "X-Tessera-Key: tk_<your-free-key>" \
  -H "Authorization: Bearer sk-<your-openai-key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hello"}]}'

# Response shape = plain OpenAI. Behind the scenes: route + cache + compress + batch.
# Savings counter ticks live at ledger.tesseraai.io/portal.
```

---

## Quality SLA — auto-rollback + 10% credit

**Quality is the primary contract; savings are the second-order effect. We address the quality story first because that's the only honest answer to "what happens when the cheaper model is dumber?"**

A canary runs your workload at 10% sample rate against the baseline model, scored by a [promptfoo](https://promptfoo.dev) eval set you can define. If a workload's stack mean-score drops below 0.95 for 3 consecutive days with at least 30 samples per stack:

1. The specific stack (e.g. `m1+m7`) auto-disables. `m1` alone and `m7` alone stay live — surgical rollback, not nuclear.
2. A 10% credit on that month's subscription lands on your account automatically.
3. An `audit_event` records the breach + credit + reactivation timeline in your `/portal/audit` ledger.

You can also flip the global kill-switch in `/portal/billing` at any time — traffic continues flowing as passthrough, just with no mutation. We treat quality regression as our problem, not yours: you get the SLA credit automatically, you don't have to file anything.

---

## Worked example — what a flat subscription looks like

**TL;DR — a customer-support agent burning $24,000/month on `gpt-4o` cuts inference cost to $9,400/month, pays a flat $999 Growth-tier subscription, and keeps 100% of the savings — $13,601/month back. Quality canary held at 0.96 across all four mechanics (above the 0.95 floor documented in the section above).**

A customer-support AI agent runs on `gpt-4o` with high prompt repetition (FAQ-style queries). 5B tokens / month at OpenAI list prices (70% input @ $2.50/M, 30% output @ $10/M) sits around **$24,000/month** at the start of the period. At ~5B gross monthly tokens this workload lands on the **Growth tier ($999/month flat)**.

After enabling Tessera (no code change beyond one-line activate):

| Stage | Cost / month | Savings |
|---|---:|---:|
| Baseline (OpenAI direct) | $24,000 | — |
| + auto-cache (35% hit rate on FAQ prefix) | $15,600 | $8,400 |
| + auto-route (`gpt-4o → gpt-4o-mini` where quality canary holds) | $11,520 | $4,080 |
| + prompt cache (OpenAI cached-input rate, 50% off cached prefix) | $10,200 | $1,320 |
| + context pruning (RAG trim) | $9,400 | $800 |
| **Tessera-optimized inference total** | **$9,400** | **$14,600 / month** |
| **Tessera subscription (Growth tier, flat)** | $999 | — |
| **Customer total pay** | **$10,399** | **$13,601 saved / month** |

You keep 100% of the measured savings; the only line we add is the flat $999 subscription. Quality canary mean-score held at 0.96 across all stages (floor 0.95) — if it had dropped, the auto-route mechanic would have rolled back automatically and the customer received a 10% SLA credit that month. **Numbers vary by workload shape.** Run your own workload free for 60M tokens to measure your actual delta.

**Verify the savings math yourself.** Every billable line is traceable back to two immutable cost figures pinned to a multi-source pricing catalog snapshot captured at request time. Two engineers, three hours, can re-derive any month from raw inputs. Full procedure at [tesseraai.io/trust](https://tesseraai.io/trust).

---

## What Tessera does to each request

Ten mechanics shipped (`m1`-`m3`, `m5`-`m11`), one more in design (`m12` PII / jailbreak / toxicity guardrails — see the [roadmap](https://tesseraai.io/how-it-works)). `m4` is intentionally skipped (no mechanic in that slot). Each mechanic is per-workload opt-in. Stage 3 mutex caps content-mutating mechanics at one per request to bound quality risk.

Audit-log chips in `/portal/audit` use the short codes in `<sub>` below — that's the bridge if you want to map a specific request back to which mechanic fired.

| Mechanic | What it does | Effect |
|---|---|---|
| **Auto-route** <sub>(m1)</sub> | Swap requested model for a cheaper-equivalent in the same family (e.g. `gpt-4o → gpt-4o-mini`) when quality canary holds ≥ 0.95 | 30–60% cost cut on most routed calls; up to 95% on the largest tier drops |
| **Auto-cache** <sub>(m2)</sub> | Exact-match KV cache for repeated prompts within TTL | Up to 100% on cache hits |
| **Auto-compress** <sub>(m3)</sub> | Whitespace and structural normalization only — preserves code fences and JSON value ranges. We never paraphrase your text. Per-role opt-in (system prompts and / or user turns) | 5–15% prompt size reduction |
| **Semantic cache** <sub>(m5)</sub> | Embedding-based similarity cache (cosine ≥ 0.95) for near-duplicate prompts | Up to 100% on semantic hits |
| **Prompt cache** <sub>(m6)</sub> | Native provider prompt-cache headers — OpenAI cached-input rate (50% off), Anthropic prompt caching (90% off cache reads) | 50–90% on cached prefixes depending on provider |
| **Context pruning** <sub>(m7)</sub> | Conversation + RAG-block aware prune when message count > 12 or body > 32 KB | 10–40% prompt token reduction |
| **Structured output** <sub>(m8)</sub> | Inject `response_format=json_schema` (strict mode) or `json_object` (auto mode) when an `expected_schema` is set | 10–35% output cost cut on JSON workloads |
| **Output ceiling** <sub>(m9)</sub> | Cap `max_tokens` from rolling p90 truncation rate per workload — eliminates wasted completion tokens on responses that always finish short | 5–15% on output-bound workloads |
| **Auto-batch** <sub>(m10)</sub> | Route async-tolerant calls to provider Batch APIs (OpenAI Batch, Anthropic Message Batches — both 50% off) | 50% on batch-eligible traffic |
| **Cross-provider failover** <sub>(m11)</sub> | Opt-in passive failover to OpenRouter when primary upstream returns 5xx / connection error / timeout. Same eval gate; gated on workload `compliance_tier` (regulated workloads skip) | Reliability, not cost — quality preservation estimate per pair |

---

## How it works (60 seconds)

1. **Get a free API key** — email + ToS at [tesseraai.io/dev](https://tesseraai.io/dev). You receive a `tk_` key (shown once) plus a magic-link for dashboard access (we use passwordless email auth — no SSO yet, on the roadmap).
2. **Install the SDK** — `pip install tessera-llm-proxy` or `npm install @tessera-llm/tessera-sdk`.
3. **One line** — `tessera.activate("tk_…")` (Python) or `activate("tk_…")` (Node).
4. **Watch the counter** — tokens used + savings number tick live on [ledger.tesseraai.io/portal](https://ledger.tesseraai.io/portal).

Provider keys (your OpenAI `sk-…`, Anthropic `sk-ant-…`, etc.) stay in your environment. Tessera forwards them upstream untouched. We never store your prompts or completions — only token counts and cost deltas. See the [data privacy FAQ](#do-you-store-my-prompts-or-completions) below.

---

## Pricing

Flat monthly subscription, priced by your **gross monthly tokens submitted** (before optimization). You keep **100% of the measured savings** — the subscription is the only line we bill.

| Tier | Gross tokens / month | Price / month |
|---|---|---:|
| **Free Sandbox** | ≤ 60M | $0 |
| **Starter** | ≤ 1B | $199 |
| **Growth** | ≤ 5B | $999 |
| **Scale** | ≤ 20B | $3,999 |
| **Enterprise** | 20B+ | Custom |

| | **Free Sandbox** | **Paid tiers** |
|---|---|---|
| Rate limit | 30 req / min | 60 req / min |
| Monthly savings statement | — | Audit-grade PDF |
| Anomaly response | Read-only alerts | Auto-throttle on cost spike, auto-halt on runaway |
| Team seats | — | Up to 5 |

The kill-switch in `/portal/billing` pauses optimization any time; traffic still flows passthrough.

Upgrade flow: start free, then pick a tier inside the dashboard once your token volume crosses the Sandbox ceiling — no separate signup. Full terms: [tesseraai.io/terms](https://tesseraai.io/terms).

---

## Supported providers

**Patched at SDK level (zero-config — `activate()` wires these up automatically):** OpenAI, Anthropic, Mistral, Groq, Cohere.

**Available via OpenAI-compatible base URL** (use `tessera.url(provider)` + `tessera.headers()` — see [`examples/direct-provider.py`](./examples/direct-provider.py)):

| Provider | Tessera route |
|---|---|
| OpenAI | `https://api.tesseraai.io/v1/openai` |
| Anthropic | `https://api.tesseraai.io/v1/anthropic` |
| Google (Gemini AI Studio) | `https://api.tesseraai.io/v1/google` |
| xAI | `https://api.tesseraai.io/v1/xai` |
| Cohere | `https://api.tesseraai.io/v1/cohere` |
| Mistral | `https://api.tesseraai.io/v1/mistral` |
| DeepSeek | `https://api.tesseraai.io/v1/deepseek` |
| Groq | `https://api.tesseraai.io/v1/groq` |
| Together AI | `https://api.tesseraai.io/v1/together` |
| Fireworks AI | `https://api.tesseraai.io/v1/fireworks` |
| OpenRouter | `https://api.tesseraai.io/v1/openrouter` |
| Perplexity | `https://api.tesseraai.io/v1/perplexity` |
| Cerebras | `https://api.tesseraai.io/v1/cerebras` |

AWS Bedrock, Azure OpenAI, Vertex AI — September 2026.

---

## Compared to LLM observability tools

| | **Tessera** | LLM observability tools |
|---|---|---|
| Position | Substrate proxy in request path | Observability sidecar |
| Optimization | Does it (route, cache, compress, batch in real time) | Surfaces charts, no real-time mutation |
| Engineer effort | Two headers, that's it | Set up tracing + dashboards |
| Billing model | Flat monthly subscription by token volume | Per-seat or per-event subscriptions |
| Co-existence | Yes — observability tools still get your telemetry downstream | Complementary |

---

## Frameworks & examples

The [examples/](./examples/) directory has runnable snippets:

- [`openai-wrap.py`](./examples/openai-wrap.py) — OpenAI client (Python)
- [`openai-wrap.ts`](./examples/openai-wrap.ts) — OpenAI client (Node)
- [`anthropic-wrap.py`](./examples/anthropic-wrap.py) — Anthropic client
- [`langchain-wrap.py`](./examples/langchain-wrap.py) — LangChain via transparent SDK patching (works because `activate()` patches the underlying OpenAI / Anthropic / etc. clients that LangChain uses internally)
- [`direct-provider.py`](./examples/direct-provider.py) — DeepSeek, Together, Fireworks, etc. via OpenAI-compatible URL

Compatible with LangChain, LlamaIndex, CrewAI, AutoGen, Mastra, Pydantic AI, and Vercel AI SDK — they all call the underlying provider SDK constructors that `activate()` patches.

---

## Framework integrations — dedicated packages

If you're already building on a framework, the dedicated integration package is the cleaner ergonomic fit than the transparent SDK patch:

| Framework | Package | Install |
|---|---|---|
| **LangChain** (Python + Node) | [`tessera-langchain`](https://github.com/tessera-llm/tessera-langchain) | `pip install tessera-langchain` · `npm install @tessera-llm/langchain` |
| **Vercel AI SDK** (Node) | [`@tessera-llm/vercel-ai`](https://github.com/tessera-llm/tessera-vercel-ai) | `npm install @tessera-llm/vercel-ai` |
| **LlamaIndex** (Python) | [`tessera-llamaindex`](https://github.com/tessera-llm/tessera-llamaindex) | `pip install tessera-llamaindex` |
| **Mastra** (Node) | [`@tessera-llm/mastra`](https://www.npmjs.com/package/@tessera-llm/mastra) | `npm install @tessera-llm/mastra` |
| **Pydantic AI** (Python) | [`tessera-pydantic-ai`](https://pypi.org/project/tessera-pydantic-ai/) | `pip install tessera-pydantic-ai` |
| **CrewAI** (Python) | [`tessera-crewai`](https://pypi.org/project/tessera-crewai/) | `pip install tessera-crewai` |
| **AutoGen 0.4+** (Python) | [`tessera-autogen`](https://pypi.org/project/tessera-autogen/) | `pip install tessera-autogen` |

All integrations use the same proxy at `api.tesseraai.io` and the same `tk_…` API key — install whichever matches your codebase. `tessera-sdk` (this package) and the framework-specific integrations are safe to use side by side.

---

## Type safety

- **Node / TypeScript:** Ships full type declarations (`dist/index.d.ts`). Autocomplete and type-checking work out of the box.
- **Python:** Ships the [PEP 561](https://peps.python.org/pep-0561/) `py.typed` marker — `mypy --strict` recognises the package as typed. All public functions are annotated; return types are explicit.

---

## Tessera in one paragraph (for search engines)

Tessera is an open-source LLM gateway and AI cost-optimization proxy for OpenAI, Anthropic, Google Gemini, Mistral, xAI, Cohere, DeepSeek, Groq, Together, Fireworks, OpenRouter, Perplexity, and Cerebras. It sits in your request path as a substrate proxy, auto-routes to cheaper-equivalent models, applies exact-match and semantic caching, compresses prompts, and batches eligible calls — all measured per request. Pricing is a flat monthly subscription priced by gross monthly token volume, with a free 60M-tokens-per-month Sandbox tier for development; customers keep 100% of the measured savings. Built for AI-native SaaS teams looking for OpenAI cost reduction without re-architecture.

---

## FAQ

### Do you store my prompts or completions?

No. The proxy logs token counts, model identifiers, cost deltas, and `request_id` — never the prompt or response bodies. The exact-match cache stores hashed prompts (sha256) for lookup keys; the semantic cache stores prompt embeddings (one-way, not invertible to recover prompt text). [Full data handling](https://tesseraai.io/security).

### How do you handle PII in prompts?

We don't inspect prompt content. The proxy forwards your request body upstream byte-for-byte to the provider you specified, then forwards their response back. Our measurement layer reads only the token counts, model identifiers, and cost deltas from headers and usage objects — never the prompt or response bodies. If your workload contains PII, your data path is identical to calling the provider directly, plus one TLS hop through Cloudflare edge.

### What happens to my OpenAI / Anthropic rate limits?

Your provider rate limits are unchanged. Tessera forwards your provider key upstream, so you stay on whatever tier your account holds. Our own rate limits (30 req/min on Free Sandbox, 60 req/min on paid tiers) apply on top — they exist to prevent abuse of the free tier and are lifted on request for paid customers with confirmed traffic spikes. Email [founder@tesseraai.io](mailto:founder@tesseraai.io).

### What happens if Tessera is down?

Your application sees an HTTP 5xx from our edge. We recommend wrapping the SDK with your existing retry / fallback logic — most LLM SDKs ship with built-in retry, keep it on. For mission-critical paths, set the proxy base URL per environment so you can flip back to the provider's direct URL with one config change. Target: 99.9% uptime.

### Do you offer self-hosted or air-gapped deployment?

Not today. The proxy runs on our hosted edge — that's how we measure savings against the canonical `pricing_catalog`. If you have a hard data-residency requirement and an annual contract size that justifies it, email [founder@tesseraai.io](mailto:founder@tesseraai.io) and we'll talk.

### What's the latency overhead?

Tessera adds ~15–40 ms p50 on cache-miss paths (one extra TLS hop to Cloudflare edge). On cache hits, **latency drops** (no upstream call). Auto-batch trades latency for cost — opt-in per workload only.

### How do you measure quality without seeing prompts?

You provide a [promptfoo](https://promptfoo.dev) golden set (5–50 prompts representative of your workload). The canary cron runs your workload's mechanic stack at 10% sample rate against the baseline model, scored by your eval set, daily. The aggregate `mean_score` per stack drives the quality SLA. You retain full control of the eval set — we just run it.

### Where are you in your SOC 2 journey?

Targeting SOC 2 Type 1 by Q3 2026. Current security posture: zero prompt or response storage at-rest, customer `Authorization` keys at-rest-encrypted via Supabase Vault (XChaCha20-Poly1305-IETF under the hood), RLS isolation per tenant, audit trail with cryptographic provenance via `pricing_catalog` snapshot ids. [Security page](https://tesseraai.io/security).

### Why open-source the SDK if the proxy is closed?

The SDK is the integration surface — the part you ship in your binary. Apache-2.0 lets you fork, audit, vendor in, and prove to your security review that nothing exotic happens client-side. The proxy at `api.tesseraai.io` is closed because the mechanic implementations are the asymmetric IP. The wire format is open — any HTTPS client speaking OpenAI / Anthropic / Google shapes can use Tessera without our SDK.

### How is the savings number computed — couldn't you inflate it?

Each request emits two cost figures: `original_cost_usd` (priced at the **requested** model's catalog rate) and `actual_cost_usd` (priced at the **actual** model after routing + cache hits + provider discounts). Both rates are pinned to a [`pricing_catalog`](https://tesseraai.io/how-it-works) snapshot version id captured at the request — immutable, with multi-source verification (LiteLLM + tokencost + OpenRouter API — all three must agree within 1%, confidence-scored). Mid-contract price changes don't retroactively alter past savings. The audit ledger is yours to export.

### Can I see which mechanics fired on a specific request?

Yes. `/portal/audit` shows a chip strip per request (`m1`, `m2`, `m3`, etc. — the same short codes used in the mechanic table above). Each `request_id` is searchable by `feature_tag` / `customer_tag` headers you can set per call. Full mechanic reference at [tesseraai.io/how-it-works](https://tesseraai.io/how-it-works).

### How do I opt out of a specific mechanic?

Per workload, in `/portal/settings`: toggle any mechanic on / off. Or set the request-level header `x-tessera-do-not-optimize: true` for one-off passthrough. The kill-switch in `/portal/billing` pauses everything across all workloads.

---

## Documentation

- **Python SDK:** [./python/README.md](./python/README.md)
- **Node SDK:** [./node/README.md](./node/README.md)
- **Architecture and mechanic reference:** [tesseraai.io/how-it-works](https://tesseraai.io/how-it-works)
- **Security + Quality SLA:** [tesseraai.io/security](https://tesseraai.io/security)
- **Engineering blog:** [tesseraai.io/blog](https://tesseraai.io/blog)
- **Discussions:** [github.com/tessera-llm/tessera-sdk/discussions](https://github.com/tessera-llm/tessera-sdk/discussions)

---

## Contributing

PRs welcome for new examples, framework adapters, type-stub improvements, and bug fixes. See [CONTRIBUTING.md](./CONTRIBUTING.md) and the [Code of Conduct](./CODE_OF_CONDUCT.md).

- **Bug reports:** [github.com/tessera-llm/tessera-sdk/issues](https://github.com/tessera-llm/tessera-sdk/issues)
- **Security:** [security@tesseraai.io](mailto:security@tesseraai.io) — see [SECURITY.md](./SECURITY.md)

---

## License

Apache-2.0. See [LICENSE](./LICENSE).

---

## About Tessera

Tessera is the **substrate layer** for **LLM cost optimization**, also called the **Optimize Layer** in our product surface. A thin proxy that sits in your application's **request-path**, applies a conservative cascade of optimization mechanics, and measures every saved dollar against an **audit-immutable** baseline. We bill a **flat monthly subscription** priced by gross monthly token volume; customers keep **100% of verified savings**. No per-token gateway fee; the category we operate in is "**LLM cost-optimization proxy**," distinct from per-token **AI gateways** and observability dashboards.

Where observability tools tell you what you spent and AI gateways re-shape the request without measuring the cost delta, Tessera is the layer that does both, and proves the savings back to you down to the dollar. The **verified-savings ledger** at [`ledger.tesseraai.io`](https://ledger.tesseraai.io) shows every original-vs-actual cost pair, snapshot-pinned to a `pricing_catalog` version captured at request time. Mid-contract price changes don't retroactively alter past savings. This is the **FinOps**-friendly model for AI inference: every line of the bill traces to a code-enforced rule.

Operated by Fintechagency OÜ (Estonia, registry code 16638667).

- Developer entry: [tesseraai.io/dev](https://tesseraai.io/dev)
- Mechanic reference: [tesseraai.io/how-it-works](https://tesseraai.io/how-it-works)
- Dashboard: [ledger.tesseraai.io](https://ledger.tesseraai.io)
- Security posture: [tesseraai.io/security](https://tesseraai.io/security)
- Engineering blog: [tesseraai.io/blog](https://tesseraai.io/blog)
- Landing: [tesseraai.io](https://tesseraai.io)
