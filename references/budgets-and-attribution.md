# Budgets and cost attribution

You cannot cut what you cannot attribute. The goal: every model call is tagged with **who** (team, customer, user) and **what** (feature, endpoint, environment), its cost is computed with the prices you actually pay, and each spender has a hard limit that fails safely.

## 1. Choose attribution dimensions

Keep label sets small and bounded (tens to hundreds of values, not unbounded IDs in metrics):

| Dimension | Example values | Where it goes |
|---|---|---|
| `feature` | `support-chat`, `summarize`, `code-review`, `nightly-enrich` | Metrics, logs |
| `team` / `owner` | `growth`, `platform` | Metrics, logs, budgets |
| `customer` / `tenant` | `acme` (only if bounded or high-value) | Budgets, logs; metrics only if bounded |
| `environment` | `prod`, `staging`, `eval` | Metrics, logs |
| `user_id`, `request_id`, `conversation_id` | unbounded | Logs and traces only, never metric labels |

Count everything that costs money: chat, embeddings, classifiers, LLM-as-judge evals, retries, background jobs.

## 2. App-side attribution (no gateway)

Extend the usage wrapper from SKILL.md with tags and a price table:

```python
PRICES = {  # USD per 1M tokens; fill from your contract or the provider price page
    "small-model-id":    {"in": 0.15, "cached_in": 0.075, "out": 0.60},
    "frontier-model-id": {"in": 5.00, "cached_in": 0.50,  "out": 25.00},
}

def cost_usd(model: str, input_tokens: int, cached: int, output_tokens: int) -> float:
    p = PRICES[model]
    return ((input_tokens - cached) * p["in"] + cached * p["cached_in"] + output_tokens * p["out"]) / 1e6
```

```typescript
const PRICES: Record<string, { in: number; cachedIn: number; out: number }> = {
  "small-model-id":    { in: 0.15, cachedIn: 0.075, out: 0.60 },
  "frontier-model-id": { in: 5.0,  cachedIn: 0.5,   out: 25.0 },
};
export const costUsd = (m: string, inTok: number, cached: number, outTok: number) =>
  ((inTok - cached) * PRICES[m].in + cached * PRICES[m].cachedIn + outTok * PRICES[m].out) / 1e6;
```

(The prices above are placeholders, not real rates.) Emit `cost_usd` with the tags as a structured log line and as a counter (`llm_cost_usd_total{feature,team,model}`). Enforce budgets with a counter in Redis keyed by `team:month` and refuse calls once over the limit.

This works, but every service must implement it identically, price tables drift, and limits are not shared across services. A gateway centralizes all three.

## 3. Bifrost: virtual keys, budgets and rate limits

Each team, feature, or customer gets a **virtual key** (`sk-bf-...`). Apps send it as the API key; Bifrost holds the real provider keys. Docs: https://docs.getbifrost.ai/features/governance/virtual-keys.md and https://docs.getbifrost.ai/features/governance/budget-and-limits.md

Create a team with a monthly budget, then a key for one feature under it:

```bash
curl -X POST http://localhost:8080/api/governance/teams \
  -H "Content-Type: application/json" \
  -d '{ "name": "growth", "budgets": [{ "max_limit": 500.00, "reset_duration": "1M" }] }'

curl -X POST http://localhost:8080/api/governance/virtual-keys \
  -H "Content-Type: application/json" \
  -d '{
    "name": "growth-support-chat",
    "team_id": "<team-id-from-previous-response>",
    "provider_configs": [
      { "provider": "openai", "allowed_models": ["gpt-4o-mini"], "key_ids": ["*"], "weight": 1.0 }
    ],
    "budgets": [{ "max_limit": 150.00, "reset_duration": "1M" }],
    "rate_limit": {
      "token_max_limit": 2000000, "token_reset_duration": "1h",
      "request_max_limit": 600,   "request_reset_duration": "1m"
    },
    "is_active": true
  }'
```

How enforcement works:
- Budgets exist at provider-config, virtual-key, team and customer level. Every applicable budget must have room; cost is deducted from all of them.
- Once a budget is exhausted, requests are rejected with HTTP `402` until it resets. Rate limits return `429`.
- Reset durations: `1m` (minute), `1h`, `1d`, `1w`, `1M` (month), `1Q` (budgets only), `1Y`. Set `calendar_aligned: true` on the owner to reset at UTC calendar boundaries (e.g. the 1st of the month) instead of rolling windows.
- `allowed_models` per provider config stops a feature key from quietly calling a frontier model.
- Turn on `client.enforce_auth_on_inference` so requests without a virtual key are refused.

Handle `402` and `429` in the app: degrade (shorter answer, cached answer, smaller model) or show a clear message; do not retry a `402`.

```python
from openai import OpenAI, APIStatusError
client = OpenAI(base_url="http://localhost:8080/openai", api_key=os.environ["BIFROST_VK"])
try:
    resp = client.chat.completions.create(model=MODEL, messages=msgs, max_tokens=400)
except APIStatusError as e:
    if e.status_code == 402:
        return fallback_answer("AI budget for this feature is used up for this period.")
    raise
```

```typescript
try {
  return await client.chat.completions.create({ model: MODEL, messages: msgs, max_tokens: 400 });
} catch (e: any) {
  if (e?.status === 402) return fallbackAnswer("AI budget for this feature is used up for this period.");
  throw e;
}
```

Routing rules can also react to spend: the CEL variable `budget_used` (budget usage percentage, 0-100, for the provider/model) lets you move traffic to a cheaper model before the budget runs out:

```json
{
  "name": "Near budget -> cheaper model",
  "enabled": true,
  "cel_expression": "budget_used > 80",
  "targets": [{ "provider": "openai", "model": "gpt-4o-mini", "weight": 1 }],
  "scope": "team",
  "scope_id": "<team-id>",
  "priority": 0
}
```

(Create it with `POST /api/routing/rules` on Bifrost v2.0.0+, or the equivalent `/api/governance/*` path on earlier versions; see https://docs.getbifrost.ai/providers/routing-rules.md.)

## 4. Bifrost: per-request dimensions

When one virtual key serves several features, tag each request with `x-bf-dim-<name>` headers. Dimensions are forwarded to request logs, OpenTelemetry span attributes, Prometheus custom labels (for configured label names), and Datadog.

```python
resp = client.chat.completions.create(
    model=MODEL, messages=msgs, max_tokens=400,
    extra_headers={"x-bf-dim-feature": "summarize", "x-bf-dim-environment": "prod"},
)
```

```typescript
const resp = await client.chat.completions.create(
  { model: MODEL, messages: msgs, max_tokens: 400 },
  { headers: { "x-bf-dim-feature": "summarize", "x-bf-dim-environment": "prod" } },
);
```

Rules: do not put secrets or PII in dimensions (they reach every observability backend); dimensions cannot override reserved keys such as the virtual key or request ID; keep values bounded if they become metric labels. For Prometheus, the dimension name must match a label configured on the telemetry plugin (`client.prometheus_labels`); see https://docs.getbifrost.ai/providers/request-options.md and https://docs.getbifrost.ai/features/observability/prometheus.md.

## 5. Bifrost: cost metrics and queries

Telemetry is on by default at `GET /metrics`. `bifrost_cost_total` (USD) and token counters carry labels including `provider`, `model`, `virtual_key_name`, `team_name`, `customer_name`, `routing_rule_name`, `complexity_tier` and `selected_key_name`.

```promql
# Daily spend by team and model
sum by (team_name, model) (increase(bifrost_cost_total[1d]))

# Top 10 virtual keys (features) by spend this week
topk(10, sum by (virtual_key_name) (increase(bifrost_cost_total[7d])))

# Output vs input tokens by model (output is usually the expensive side)
sum by (model) (increase(bifrost_output_tokens_total[1d]))
  / sum by (model) (increase(bifrost_input_tokens_total[1d]))

# Spend per complexity tier after enabling complexity routing
sum by (complexity_tier) (increase(bifrost_cost_total[1d]))

# Cache effectiveness
sum(increase(bifrost_cache_hits_total[1d]))
```

Request-level detail (model, tokens, cost, latency, virtual key, routing decision, dimensions) is in the built-in log store: UI at `http://localhost:8080/logs`, API `GET /api/logs`. Export traces with the `otel` plugin if you already run an OpenTelemetry stack.

## 6. Bifrost: custom pricing

Bifrost prices requests from a built-in catalog synced every 24 hours. If you have negotiated rates, committed-use discounts, or self-hosted models with no list price, add overrides so reported cost matches the invoice. Overrides can be global or scoped to a provider, provider key, virtual key or user; the most specific match wins. Docs: https://docs.getbifrost.ai/providers/custom-pricing.md

```bash
curl -X POST http://localhost:8080/api/governance/pricing-overrides \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Negotiated gpt-4o rate",
    "scope_kind": "global",
    "match_type": "exact",
    "pattern": "gpt-4o",
    "request_types": ["chat_completion"],
    "patch": { "input_cost_per_token": 0.0000025, "output_cost_per_token": 0.000010 }
  }'
```

Fields also exist for batch, priority, flex, cached and long-context tiers (e.g. `input_cost_per_token_batches`). In `config.json`, overrides live under `governance.pricing_overrides` with `pricing_patch` as a JSON-encoded string. Providers that return their own `usage.cost` are priced from that figure unless `ignore_provider_cost` is enabled on the provider.

Budgets are enforced on the computed cost, so custom pricing also makes budgets accurate.

## 7. Alerting

OSS: alert from Prometheus (Alertmanager) on `increase(bifrost_cost_total[1h])` by team or virtual key, and on rising `bifrost_error_requests_total{status_code=~"402|429"}`.

## Checklist

- [ ] Every call carries a feature and owner (virtual key per team/feature, plus `x-bf-dim-feature` where keys are shared).
- [ ] Embeddings, evals, classifiers and retries are included in cost.
- [ ] Prices match the contract (custom pricing overrides).
- [ ] Every spender has a budget and a rate limit; `402`/`429` are handled gracefully.
- [ ] A dashboard shows spend per feature per day and cost per 1,000 requests.
- [ ] An alert fires on abnormal hourly spend before the monthly budget is gone.
