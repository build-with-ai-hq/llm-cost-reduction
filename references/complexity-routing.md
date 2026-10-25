# Complexity-based model routing

Goal: send each request to the cheapest model that can handle it. Simple requests (lookups, reformatting, short factual answers) go to a small model; complex ones (multi-step reasoning, hard debugging, design) go to a frontier model. This matters most for endpoints that receive mixed traffic, such as chat boxes, copilots and coding assistants, where one static model choice is always either too expensive or too weak.

## Before routing: know your traffic

1. Sample 200-500 real requests from the endpoint (strip PII).
2. Label each one SIMPLE / MEDIUM / COMPLEX by hand, or have a strong model label them and spot-check.
3. Run the candidate cheap model on the SIMPLE bucket and the mid-tier model on the MEDIUM bucket. Score against the frontier model's answers or a rubric.
4. Estimate savings: `share_simple x (price_frontier - price_small) + share_medium x (price_frontier - price_mid)`, minus classifier cost.

If 70% of traffic is SIMPLE and the small model holds quality on it, routing is worth doing. If almost everything is COMPLEX, spend the effort on prompt trimming and caching instead.

## Option A: Bifrost Complexity Router (recommended, no app changes)

Bifrost classifies each request into `SIMPLE`, `MEDIUM` or `COMPLEX` and publishes the tier as `complexity_tier` to its CEL routing rules. The application keeps calling the gateway with the model it uses today; a matching rule overrides provider and model. Docs: https://docs.getbifrost.ai/features/complexity-router.md and https://docs.getbifrost.ai/providers/routing-rules.md

Key behaviors:
- Classification runs only when some routing rule references `complexity_tier`.
- If the classifier is unavailable, times out, or produces no tier, the complexity rule does not match and the request continues through normal routing (the app's requested model). Failure is safe.
- The Router only sets the variable; you write one rule per tier you want to route.
- Classifier cost counts toward the request's cost and budgets under the model that answered.

### Step 1: Pick a classifier

| | Decision model | Semantic |
|---|---|---|
| How it decides | A small decision model judges the work against tier definitions, signals and examples | Embeds the latest user message and takes the tier of the nearest reference phrase |
| You maintain | Nothing to start; optionally tune definitions/signals/examples | Reference phrases per tier (150 shipped, up to 750) |
| Requires | A provider serving the decision model (Typesafe Jev by default; also OpenAI, OpenRouter, Cloudflare, Perplexity, Ollama, Azure, Databricks, or self-hosted) | An embedding provider/model with an enabled key; optionally a shared vector store |
| Added latency | One decision call per classified request (Jev typically 600-800 ms; default timeout 1.5 s) | One embedding call plus a vector lookup, usually much faster |
| Best when | Traffic is varied and you do not want to curate data | Traffic is predictable and latency matters most |

You can combine them: Semantic as primary with the decision model as fallback (`semantic.fallback: "decision"`) for requests no phrase matches confidently.

Configure in the UI under **Models > Complexity Router**, via `PUT /api/routing/complexity-analyzer-config`, or in `config.json` under `governance.complexity_analyzer_config`. The `/api/routing/*` endpoints exist in Bifrost v2.0.0 and above; earlier versions use `/api/governance/*`.

Decision-model primary (`config.json`):

```json
{
  "governance": {
    "complexity_analyzer_config": {
      "classifier": "decision",
      "decision": {
        "provider": "typesafe",
        "model": "jev-latest",
        "previous_message_count": 1,
        "timeout": "1.5s"
      },
      "session": { "enabled": true },
      "keywords": {
        "simple_keywords": ["fix the grammar in this sentence."],
        "medium_keywords": ["apply a standard method to this technical question."],
        "complex_keywords": ["design a solution under several competing constraints."]
      }
    }
  }
}
```

Semantic primary with OpenAI embeddings:

```json
{
  "governance": {
    "complexity_analyzer_config": {
      "classifier": "semantic",
      "semantic": {
        "provider": "openai",
        "embedding_model": "text-embedding-3-small",
        "timeout": "1.5s",
        "min_similarity": 0,
        "message_history_count": 1,
        "count_toward_budgets": false,
        "vector_store": "embedded",
        "fallback": "none"
      },
      "session": { "enabled": true },
      "keywords": {
        "simple_keywords": ["what is a mutex?", "fix the grammar in this sentence."],
        "medium_keywords": ["add api-key auth: hash the keys, reject revoked ones, and never log them."],
        "complex_keywords": ["balance testing, prescribing rules, and staffing against rising resistant infections."]
      }
    }
  }
}
```

Notes that trip people up:
- The `keywords` lists (reference phrases) are required in a saved config even when the decision model is primary. They are unused by the decision model.
- The update API replaces the whole configuration. Leaving out `decision.criteria` resets tier guidance to the shipped defaults. To change one field, `GET` the config, edit, `PUT` it back.
- `GET /api/routing/complexity-analyzer-status` reports semantic warmup state (`warming`, `ready`, `failed`) and returns the shipped decision-model guidance as `decision_defaults`.
- With several Bifrost nodes and the Semantic classifier, use a shared vector store (`"vector_store": "vector_store"`) so each node does not re-embed phrases on restart.

### Step 2: Add routing rules

Start with a single carve-out, validate it, then add tiers. Rules live in the UI (**Routing Rules**, with a visual CEL builder), the API (`POST /api/routing/rules`), or `governance.routing_rules` in `config.json`. Rules are first-match-wins within a scope, ordered by `priority` (lower first; each priority unique per scope). Target `weight`s in one rule must sum to 1.

Cheapest first move, sending SIMPLE traffic to a small model while everything else stays as is:

```bash
curl -X POST http://localhost:8080/api/routing/rules \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Simple -> small model",
    "enabled": true,
    "cel_expression": "complexity_tier == \"SIMPLE\"",
    "targets": [{ "provider": "openai", "model": "gpt-4o-mini", "weight": 1 }],
    "fallbacks": ["anthropic/claude-haiku-4-5"],
    "scope": "global",
    "priority": 0
  }'
```

Full ladder (`config.json`); substitute the models your project already uses:

```json
{
  "governance": {
    "routing_rules": [
      {
        "id": "complexity-simple",
        "name": "Simple -> fast model",
        "enabled": true,
        "cel_expression": "complexity_tier == \"SIMPLE\"",
        "targets": [{ "provider": "groq", "model": "llama-3.1-8b-instant", "weight": 1 }],
        "scope": "global",
        "priority": 0
      },
      {
        "id": "complexity-medium",
        "name": "Medium -> balanced model",
        "enabled": true,
        "cel_expression": "complexity_tier == \"MEDIUM\"",
        "targets": [{ "provider": "openai", "model": "gpt-4o-mini", "weight": 1 }],
        "scope": "global",
        "priority": 1
      },
      {
        "id": "complexity-complex",
        "name": "Complex -> frontier model",
        "enabled": true,
        "cel_expression": "complexity_tier == \"COMPLEX\"",
        "targets": [{ "provider": "anthropic", "model": "claude-opus-4-5", "weight": 1 }],
        "scope": "global",
        "priority": 2
      }
    ]
  }
}
```

Rules in `config.json` load at startup; changes need a restart. API and UI changes apply live.

Combine the tier with other context to limit blast radius:

```cel
complexity_tier == "SIMPLE" && headers["x-feature"] == "support-chat"
request_type == "chat_completion" && complexity_tier != "COMPLEX"
team_name == "ml-research" && complexity_tier == "COMPLEX"
```

Pilot on one team by setting `"scope": "team", "scope_id": "<team-id>"` before going global. Scope order is virtual key, then team, then customer, then global.

If requests use a virtual key, the rule's target provider/model must be allowed by that key's `provider_configs`, otherwise governance refuses the attempt with `403`.

### Step 3: Session-aware routing for agents and chats

Model switches mid-conversation hurt provider prompt-cache reuse and can change answer style. With `"session": {"enabled": true}`, a session keeps the highest tier it has reached (it can go up, never down) for 24 hours of inactivity. Session ID comes from `x-bf-session-id`, or automatically from Claude Code (`x-claude-code-session-id`) and Codex headers.

Session-aware routing keeps the **tier** stable, not the provider or key. Add session affinity (same `x-bf-session-id`) to also keep the provider and key, so the conversation keeps hitting a warm prompt cache: https://docs.getbifrost.ai/providers/session-affinity.md

### Step 4: Verify and tune

- Logs (UI `/logs`, or API) record `complexity_tier`, `complexity_mechanism` (`semantic`, `decision`, `llm`, `session`, `skipped`) and, for Semantic, `complexity_score`:

  ```bash
  curl "http://localhost:8080/api/logs?complexity_tiers=SIMPLE&complexity_mechanisms=decision"
  ```

- Prometheus metrics carry `complexity_tier`, `complexity_mechanism` and `routing_rule_name` labels:

  ```promql
  sum by (complexity_tier, model) (increase(bifrost_cost_total[1d]))
  sum by (complexity_tier) (rate(bifrost_success_requests_total[1h]))
  sum by (complexity_mechanism) (rate(bifrost_success_requests_total{complexity_mechanism="skipped"}[1h]))
  ```

  A high `skipped` share means the classifier is timing out or not ready; those requests fall through to the default model and save nothing.

- Review a sample of SIMPLE-routed requests weekly. If hard requests land in SIMPLE, tune:
  - Decision model: edit tier `definition`, `signals`, `examples` under `decision.criteria.<TIER>`. A list replaces the default list, so include the defaults you want to keep. Describe the work, not the topic ("change must stay consistent across several files", not "mentions Kubernetes"). Keep guidance short; it is sent with every classification.
  - Semantic: add short reference phrases taken from real prompts to the right tier; balance questions and imperatives across tiers; raise `min_similarity` and add a fallback if accidental matches appear (the routing decision log shows the matched phrase).

- Account for classifier cost: decision calls are billed to the request; Semantic embedding spend is in `bifrost_routing_embedding_cost_total`.

## Option B: app-side router (no gateway)

If you cannot add a gateway, route in code. Keep the router cheap and fail toward quality.

```python
ROUTES = {"SIMPLE": "small-model-id", "MEDIUM": "mid-model-id", "COMPLEX": "frontier-model-id"}

CLASSIFY_PROMPT = (
    "Classify the user's request by the work it needs. "
    "SIMPLE: answerable in one step from the text or common knowledge. "
    "MEDIUM: needs subject knowledge or a few steps. "
    "COMPLEX: multi-stage reasoning, design, or hard debugging. "
    "Reply with one word."
)

def classify(text: str) -> str:
    try:
        r = client.chat.completions.create(
            model=ROUTES["SIMPLE"], max_tokens=3, temperature=0, timeout=1.5,
            messages=[{"role": "system", "content": CLASSIFY_PROMPT},
                      {"role": "user", "content": text[-4000:]}])
        tier = r.choices[0].message.content.strip().upper()
        return tier if tier in ROUTES else "COMPLEX"
    except Exception:
        return "COMPLEX"  # fail toward quality
```

```typescript
const ROUTES = { SIMPLE: "small-model-id", MEDIUM: "mid-model-id", COMPLEX: "frontier-model-id" } as const;
type Tier = keyof typeof ROUTES;

async function classify(text: string): Promise<Tier> {
  try {
    const r = await client.chat.completions.create(
      { model: ROUTES.SIMPLE, max_tokens: 3, temperature: 0,
        messages: [{ role: "system", content: CLASSIFY_PROMPT }, { role: "user", content: text.slice(-4000) }] },
      { timeout: 1500 });
    const tier = r.choices[0].message.content?.trim().toUpperCase() as Tier;
    return tier in ROUTES ? tier : "COMPLEX";
  } catch { return "COMPLEX"; }
}
```

Downsides compared with gateway routing: every service must embed the router, rules change only with a deploy, and there is no shared view of tier distribution and cost per tier unless you build it.

## Pitfalls

- Routing by prompt length or keywords. Length does not predict difficulty; a short question can be hard.
- No eval before switching SIMPLE traffic to a small model.
- Letting a conversation bounce between models every turn (use session-aware routing plus session affinity).
- Ignoring classifier latency on latency-sensitive endpoints; prefer Semantic or scope routing to endpoints where a few hundred ms is acceptable.
- Forgetting that a request the classifier skips uses the app's requested model; if that is the frontier model, a broken classifier quietly removes all savings. Watch the `skipped` share.
