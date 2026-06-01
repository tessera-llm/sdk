# Tessera SDK — Architecture

Why the SDK is intentionally thin, what the proxy actually does, and how the closed parts of the system are verifiable from the open parts.

---

## TL;DR

- The **SDK is thin (~310 lines Node, ~322 lines Python)** because that's the right shape for a transport layer. Constructor-patching plus URL + headers helpers is the whole API surface. More lines would be wrong abstraction, not more product.
- The **proxy is closed** because the routing / caching / compression / batching mechanic implementations and the cross-customer eval ledger are the asymmetric IP. The wire format is open (OpenAI / Anthropic shapes), so there is no lock-in: point your client back at `api.openai.com` and you lose the mechanic stack but nothing else.
- The **savings claim is verifiable** without trusting the proxy. Every request emits two cost figures pinned to a `pricing_catalog` snapshot. The catalog is multi-source verified (LiteLLM + tokencost + OpenRouter). The audit ledger exports as CSV from `/portal/audit`. Two engineers, three hours, can re-derive any month from raw inputs.

---

## What is open vs closed

| Layer | Status | Why |
|---|---|---|
| SDK (this repo) | Apache-2.0 | Constructor patching + URL routing + lifecycle helpers. Thin by design. |
| Wire format | Open OpenAI / Anthropic shapes | Zero lock-in. Standard JSON. |
| Audit ledger | Open evidence (CSV export, per-row provenance) | See "Verification" below. |
| Pricing catalog | Multi-source open inputs | LiteLLM `model_prices_and_context_window.json` (Apache-2.0), `tokencost` lib, OpenRouter API. All three must agree within 1% for a row to be eligible (confidence ≥ 0.95). |
| Proxy mechanic implementations | Closed | M1 router decision tree, M5 semantic-cache embedding model, M6 prompt-cache injection logic, M7 context pruner, M10 batch dispatcher. Implementation details are the asymmetric IP. Behaviour is observable through the audit ledger. |
| Cross-customer eval ledger | Closed (opt-in) | Aggregated quality signals that improve routing decisions over time. Per-customer opt-in via `contributes_to_cross_customer_intelligence` flag in `/portal/billing`. |

---

## The "thin SDK" rationale

The SDK does three things, full stop:

1. **Patches provider client constructors** — when you call `new OpenAI(...)` after `activate("tk_...")`, the constructed instance has `baseURL` set to `https://api.tesseraai.io/v1/openai` and an `X-Tessera-Key` header attached. Caller-set baseURL wins (we never override an explicit user choice).
2. **Provides `url(provider)` + `headers()` helpers** — for providers without a Node/Python SDK (DeepSeek, Together, Fireworks, OpenRouter, Perplexity, Cerebras, xAI), you wire an OpenAI-shaped client at those endpoints manually using these helpers.
3. **Lifecycle: `activate` / `deactivate` / `status` / `isActive`** — module-level singleton state, idempotent re-activation, full restore on deactivate.

That is the whole API surface. The proxy is what makes the routing / caching / compression / batching decisions. The SDK just delivers requests there.

### Why "300 lines of constructor patching" is the right size

A common review reaction: "your SDK is just 300 lines, where's the actual product?"

The honest answer: **the SDK is supposed to be 300 lines**. Shipping 3,000 lines of mechanic logic in the SDK would mean:

1. Cross-customer eval ledger would not exist (each customer's mechanic state would be isolated client-side).
2. Mechanic updates would require SDK re-install instead of a server-side rollout.
3. Auto-rollback on quality regression would be impossible (no central control plane to trigger).
4. The pricing catalog could not be multi-source verified live (clients cannot audit each other's snapshots in real time).

The SDK is a transport layer. The work happens in the proxy. Same shape as Stripe SDK + Stripe Dashboard, or AWS CLI + AWS control plane. Closed control plane is not the bug here, it's the load-bearing design choice.

---

## Verification — how to trust the savings number without trusting the proxy

Every billable request emits two cost figures in the audit ledger:

- `original_cost_usd` — the cost at the **requested model's** catalog rate, computed against the pricing catalog snapshot pinned to this request.
- `actual_cost_usd` — the cost after route + cache + compression + provider discounts.

The savings figure (`original - actual`) is recomputable from those two columns independently — you do not have to trust our aggregation.

### Multi-source pricing catalog

A pricing row enters the catalog only if all three sources agree within 1%:

1. **LiteLLM** — `model_prices_and_context_window.json` in `BerriAI/litellm` (Apache-2.0). De facto industry source-of-truth.
2. **tokencost** — Python lib maintained by AgentOps. Independent ingest of provider docs.
3. **OpenRouter** — live API pricing for cross-provider models. Independent operator.

Confidence band:
- All three agree within 1%: confidence ≥ 0.95.
- Two of three agree: confidence ≥ 0.80, row flagged for review.
- Disagreement: row not eligible for billing math until reconciled.

Snapshot id is captured at the request's mint time. Price changes mid-period do not retroactively change billing math — every request points at the catalog state in effect when it ran.

### Audit ledger CSV export

`/portal/audit` exports the full ledger as CSV. Columns include:

```
request_id, occurred_at, model_requested, model_routed, input_tokens, output_tokens,
mechanics_stack, pricing_snapshot_id, original_cost_usd, actual_cost_usd
```

### Re-derivation procedure (two engineers, three hours)

1. Pull the audit CSV for the period.
2. Pull the corresponding pricing catalog snapshots by `pricing_snapshot_id` (publicly fetchable, immutable).
3. Recompute `original_cost_usd` from raw token counts and catalog rates — should match the audit row to 4 decimal places.
4. Recompute `actual_cost_usd` from token counts + applied `mechanics_stack` discounts (per-mechanic discount math is documented at `tesseraai.io/security`). Should match within rounding.

That is the audit-immutable claim cashed out: the proxy cannot retroactively change either figure, the snapshots are immutable, the math is reproducible by a third party with read-only access to your account.

---

## Thread safety and concurrency

The SDK uses module-level singleton state (`state` in `node/src/index.ts`, `_state` in `python/tessera/core.py`). This is correct for the target runtimes:

- **Node.js single-threaded event loop**: there are no shared-state race conditions between concurrent requests on the same module. If your application uses Worker Threads, each worker gets an isolated module instance with its own state.
- **Python GIL + asyncio cooperative scheduling**: the same property holds. Module globals are not concurrently mutated by parallel callers in either CPython threading or asyncio.

The patcher install (`activate()`) is **idempotent**: calling `activate()` twice with new args correctly deactivates the prior state before patching with the new state. `deactivate()` is **safe when inactive** (no-ops cleanly).

Concurrency on the proxy side is handled by Cloudflare Workers with per-request isolation by design — request handlers do not share mutable state.

---

## Test coverage

| Component | Test framework | Test count | File |
|---|---|---|---|
| Node SDK | vitest | 15 unit tests | `node/test/index.test.ts` |
| Python SDK | pytest | 19 unit tests | `python/tests/test_core.py` |

Tests cover pure helpers (`url`, `headers`), lifecycle (`activate` / `deactivate` / `status` / `isActive`), env-var fallback, idempotent re-activation, and constructor patching against stub provider classes (no real provider SDK installed for unit tests).

Both suites run before publish via `prepublishOnly` (npm) and CI gate (PyPI). `npm test` and `pytest -q` should both stay green on `main`.

---

## Versioning

Pre-1.0 semver: minor releases may include breaking changes. Pin a floor and ceiling in production:

```
"tessera-llm-proxy>=0.1.0,<0.2"
"@tessera-llm/tessera-sdk@^0.1.0"
```

The SDK tracks the same `0.x.y` line as the proxy worker. Major bumps come with `CHANGELOG.md` entries calling out the breaking change.

---

## See also

- [README.md](./README.md) — quickstart, worked example, FAQ
- [tesseraai.io/trust](https://tesseraai.io/trust) — full verification procedure
- [tesseraai.io/security](https://tesseraai.io/security) — data handling and SOC 2 status
- [github.com/tessera-llm/mcp-server](https://github.com/tessera-llm/mcp-server) — Model Context Protocol server (sigstore SLSA v1 attested)
