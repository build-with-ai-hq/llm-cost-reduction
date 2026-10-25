# llm-cost-reduction

[![skills.sh](https://skills.sh/b/build-with-ai-hq/llm-cost-reduction)](https://skills.sh/build-with-ai-hq/llm-cost-reduction)

An Agent Skill that finds and cuts LLM and AI API spend in applications and agents, measuring first and holding output quality steady, with Python and TypeScript fixes.

## Problems this solves

- A surprise provider bill, and no way to tell which feature, team or customer caused it.
- Every request, including trivial ones, going to a frontier model.
- Repeated and near-duplicate prompts paying full price every time.
- Bloated system prompts, tool schemas, conversation history and RAG context.
- Unset `max_tokens`, uncapped agent loops and retry storms.
- Provider prompt caching available but never hitting.
- Offline and bulk jobs running on the synchronous API instead of discounted Batch APIs.
- Embeddings recomputed for unchanged content.
- No budgets or rate limits per team, feature or customer.

## Install

```bash
npx skills add build-with-ai-hq/llm-cost-reduction
```

## What the skill does

When an agent loads this skill, it:

1. Runs a grep audit to find every model call site, the models in use, missing output limits, uncapped loops, retries, unrecorded usage and prompt-cache killers.
2. Instruments calls so cost is known per feature and per request, then ranks features by spend.
3. Applies fixes in order of impact: right-sizing models with an eval set, output and loop caps, prompt and context trimming, provider prompt caching with stable prefixes, exact and semantic response caching, Batch APIs, embedding reuse, and stream cancellation.
4. Moves the controls that work best centrally to an open-source gateway (Bifrost): the Complexity Router to send simple requests to a cheap model and complex ones to a frontier model with no app changes, semantic caching, session affinity and auto prompt caching for provider cache hits, virtual keys with budgets and rate limits, `x-bf-dim-*` cost attribution, `bifrost_cost_total` metrics, and custom pricing.
5. Verifies that cost per 1,000 requests dropped, quality held on the eval set, caches hit, and caps actually stop runaway runs.

## Contents

```
SKILL.md                               Core workflow: diagnosis, fixes by impact, gateway controls, verification, common mistakes
references/complexity-routing.md       Per-request model routing: Bifrost Complexity Router setup, routing rules, rollout, tuning; app-side alternative
references/caching.md                  Provider prompt caching per provider, prefix stability, session affinity, auto prompt caching, response and embedding caches
references/budgets-and-attribution.md  Attribution dimensions, virtual keys, budgets and rate limits, x-bf-dim headers, PromQL, custom pricing, alerting
```

## License

MIT. See [LICENSE](LICENSE).
