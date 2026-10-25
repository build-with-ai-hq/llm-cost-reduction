# Caching for LLM cost reduction

Three different caches, often confused:

| Cache | What is reused | Saves | Where |
|---|---|---|---|
| Provider prompt cache | The provider's processed prompt prefix | Input-token price on the cached prefix (large discount) and latency; output still billed | Provider side; you control prefix stability |
| Response cache (exact or semantic) | The whole previous answer | The entire call | Your app or the gateway |
| Embedding cache | Vectors for text you already embedded | Embedding calls | Your app / vector DB |

## 1. Provider prompt caching

### How the providers differ (check current docs and pricing before relying on numbers)

- **OpenAI:** automatic for prompts above a minimum length (around 1,024 tokens). Cached input tokens are billed at a discount. Reported in `usage.prompt_tokens_details.cached_tokens`. A `prompt_cache_key` request parameter can improve hit rates when many requests share a prefix.
- **Anthropic:** explicit. Mark cache breakpoints with `cache_control: {"type": "ephemeral"}` on system blocks, tools, or messages (a small number of breakpoints per request; minimum cacheable length applies). Cache writes cost more than normal input, cache reads cost a small fraction; the default TTL is short (minutes) with a longer-TTL option. Reported in `usage.cache_creation_input_tokens` and `usage.cache_read_input_tokens`.
- **Google Gemini:** implicit caching on recent models, plus explicit context caching (a cached-content resource with a TTL and a storage charge) for large shared context.
- **AWS Bedrock:** prompt caching with cache checkpoints on supported models.

A cache write that is never read costs more than no cache at all on providers that charge a write premium. Cache only prefixes that repeat within the TTL.

### Make prefixes cacheable

Order every request as: tools -> system prompt -> static reference material / few-shot examples -> conversation history -> latest user message and per-request data.

Checklist:
- No timestamps, request IDs, user names, A/B flags or random example selection in the system prompt. Move per-request values into the last user message.
- Deterministic tool list order and JSON serialization (`json.dumps(..., sort_keys=True)`, stable key order in TS objects).
- Same model, same provider, same key (or org) for the whole conversation. Load balancing a session across keys or providers sends each turn to a cold cache.
- Do not rewrite earlier history (e.g. re-summarizing every turn) unless you accept a cache miss on everything after the change. Summarize in larger, infrequent steps.
- RAG chunks change per query; put them after the stable prefix, not inside the system prompt.

### Anthropic example

```python
import anthropic
client = anthropic.Anthropic()

resp = client.messages.create(
    model=MODEL,
    max_tokens=800,
    system=[
        {"type": "text", "text": LONG_STATIC_INSTRUCTIONS_AND_DOCS,
         "cache_control": {"type": "ephemeral"}},
    ],
    messages=history + [{"role": "user", "content": f"{user_msg}\n\n(current time: {now_iso})"}],
)
u = resp.usage
print(u.cache_creation_input_tokens, u.cache_read_input_tokens, u.input_tokens)
```

```typescript
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic();

const resp = await client.messages.create({
  model: MODEL,
  max_tokens: 800,
  system: [{ type: "text", text: LONG_STATIC_INSTRUCTIONS_AND_DOCS, cache_control: { type: "ephemeral" } }],
  messages: [...history, { role: "user", content: `${userMsg}\n\n(current time: ${nowIso})` }],
});
console.log(resp.usage.cache_creation_input_tokens, resp.usage.cache_read_input_tokens);
```

### OpenAI example

Nothing to enable; make the prefix stable and check the counter.

```python
resp = client.chat.completions.create(
    model=MODEL, max_tokens=500,
    messages=[{"role": "system", "content": STATIC_SYSTEM_PROMPT}, *history,
              {"role": "user", "content": user_msg}],
)
print(resp.usage.prompt_tokens_details.cached_tokens, "of", resp.usage.prompt_tokens)
```

### Diagnosing a zero hit rate

1. Diff the first ~2,000 tokens of two consecutive requests. Any difference before the end of the intended prefix is the cause.
2. Check the prefix is above the provider's minimum cacheable length.
3. Check requests in the same conversation hit the same provider, model and key.
4. Check the gap between requests is shorter than the cache TTL.

### Keep sessions on one provider and key with Bifrost

When a gateway load-balances across keys or providers, it can scatter a conversation across caches. Bifrost session affinity binds a session to the provider and key that last served it, so later turns land on the warm cache. Send a stable session ID per conversation or agent run:

```python
client = OpenAI(base_url="http://localhost:8080/openai", api_key="sk-bf-your-virtual-key")
resp = client.chat.completions.create(
    model=MODEL, messages=msgs, max_tokens=500,
    extra_headers={"x-bf-session-id": conversation_id},
)
```

```typescript
const resp = await client.chat.completions.create(
  { model: MODEL, messages: msgs, max_tokens: 500 },
  { headers: { "x-bf-session-id": conversationId } },
);
```

- Default binding TTL is one hour, refreshed on each served request; override per request with `x-bf-session-ttl` (e.g. `30m`).
- Claude Code, Codex CLI and OpenCode session headers are adopted automatically.
- Affinity applies when routing offers more than one provider or key; a request that names its provider (`openai/gpt-4o`) is never reordered across providers but still gets key-level affinity.
- Disable for a request with `x-bf-session-affinity: off`.

Docs: https://docs.getbifrost.ai/providers/session-affinity.md . If you also use the Complexity Router, enable its session-aware routing so the tier (and therefore the model) does not change mid-conversation.

Caller-supplied `cache_control` markers are forwarded unchanged. Confirm hits with the usage counters above on responses that pass through the gateway.

### Auto-inject cache breakpoints with Bifrost

Clients that send no cache markers (many agent harnesses, Codex, generic OpenAI-compatible code calling Claude) never get Anthropic-style prompt caching, so every turn pays full input price for a prefix that barely changed. Bifrost's Auto Prompt Caching marks the first cacheable block of requests that arrive without markers, per provider:

```json
{
  "providers": {
    "anthropic": {
      "keys": [{ "name": "anthropic-key-1", "value": "env.ANTHROPIC_API_KEY", "models": ["*"], "weight": 1.0 }],
      "prompt_cache": { "auto_inject": true }
    }
  }
}
```

- Off by default. Caller markers always win; a request that already has markers is forwarded unchanged.
- Injected only for models that accept explicit markers (Claude on Anthropic, Vertex, Bedrock and OpenRouter; Amazon Nova; newer OpenAI models that take `prompt_cache_breakpoint`). Implicit-caching providers are left alone.
- Optional `"ttl": "1h"` requests the longer cache lifetime (the only accepted value; omit for the provider default). Longer TTL carries a higher write price on Anthropic, so use it only when turns are minutes apart.
- `cache_control_injection_points` targets specific messages instead of the first block.
- Per-request override: `x-bf-prompt-cache-auto-inject: true|false`.
- Also settable in the UI (Model Providers > provider > Prompt Caching tab) or via `PUT /api/providers/{provider}` (that endpoint replaces the provider-level config, so send existing network settings along).

Docs: https://docs.getbifrost.ai/features/prompt-caching.md . Pair it with session affinity so the cache that was written is the one that gets read.

## 2. Response caching

### When it is safe

- Answers that depend only on the prompt: FAQ, documentation Q&A, product lookups, classification of repeated inputs, deterministic transforms, eval reruns.
- Not safe without a per-user key: anything using private user data, account state, or time-sensitive facts.
- Not safe by default: responses containing tool calls (replaying one can run a tool with someone else's arguments).

### App-side exact cache

```python
import hashlib, json, redis
r = redis.Redis()

def cached_call(feature: str, ttl: int = 3600, **kwargs):
    key = "llm:" + hashlib.sha256(json.dumps(kwargs, sort_keys=True).encode()).hexdigest()
    if (hit := r.get(key)) is not None:
        return json.loads(hit)
    resp = llm_call(feature, **kwargs)  # instrumented wrapper from SKILL.md
    if not resp.choices[0].message.tool_calls:
        r.setex(key, ttl, resp.model_dump_json())
    return json.loads(resp.model_dump_json())
```

```typescript
import { createHash } from "node:crypto";
import { createClient } from "redis";
const redis = await createClient().connect();

export async function cachedCall(feature: string, params: any, ttlSec = 3600) {
  const key = "llm:" + createHash("sha256").update(JSON.stringify(params)).digest("hex");
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit);
  const resp = await llmCall(feature, params);
  if (!resp.choices[0].message.tool_calls) await redis.set(key, JSON.stringify(resp), { EX: ttlSec });
  return resp;
}
```

Use `temperature: 0` for cacheable calls so a cached answer is representative. Note that `JSON.stringify` key order follows insertion order; build params consistently.

### Gateway cache with Bifrost (exact + semantic)

Bifrost's `semantic_cache` plugin does a direct hash lookup first, then (optionally) a semantic vector search on a miss. Docs: https://docs.getbifrost.ai/features/semantic-caching.md

Start a vector store locally:

```bash
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest
```

`config.json`:

```json
{
  "vector_store": {
    "enabled": true,
    "type": "redis",
    "config": { "addr": "localhost:6379" }
  },
  "plugins": [
    {
      "enabled": true,
      "name": "semantic_cache",
      "config": {
        "provider": "openai",
        "embedding_model": "text-embedding-3-small",
        "dimension": 1536,
        "ttl": "5m",
        "threshold": 0.8,
        "conversation_history_threshold": 3,
        "cache_by_model": true,
        "cache_by_provider": true
      }
    }
  ]
}
```

Weaviate, Qdrant and Pinecone are also supported. For exact-match only (no embedding calls), set `"dimension": 1` and omit `provider` and `embedding_model`.

Caching engages only on requests that carry a cache key:

```python
resp = client.chat.completions.create(
    model=MODEL, messages=msgs, max_tokens=300, temperature=0,
    extra_headers={"x-bf-cache-key": "support-faq", "x-bf-cache-ttl": "1h"},
)
```

```typescript
const resp = await client.chat.completions.create(
  { model: MODEL, messages: msgs, max_tokens: 300, temperature: 0 },
  { headers: { "x-bf-cache-key": "support-faq", "x-bf-cache-ttl": "1h" } },
);
```

Per-request controls: `x-bf-cache-ttl`, `x-bf-cache-threshold` (can only make matching stricter), `x-bf-cache-type: direct|semantic`, `x-bf-cache-no-store: true` (read but do not write). A `default_cache_key` in the plugin config turns caching on for requests without the header; leave it empty unless every request through that gateway is safe to cache.

Safety defaults worth knowing:
- Entries are scoped to the requesting virtual key (and user, where resolved) on top of the cache key, so two virtual keys never share entries unless `disable_identity_scoping` is set.
- Tool-call responses are not cached unless `cache_tool_call_responses: true`.
- Conversations longer than `conversation_history_threshold` messages are not cached (long chats rarely repeat exactly).

Verify: responses carry `x-bifrost-cache-hit`, `x-bifrost-cache-hit-type` (`direct` / `semantic`) and, on semantic hits, `x-bifrost-cache-similarity`. Prometheus exposes `bifrost_cache_hits_total`. Clear entries for a key with `DELETE /api/cache/clear-by-key/{cacheKey}`.

Threshold tuning: start at 0.8-0.9 for FAQ-style traffic; sample semantic hits with their similarity and the original vs. new question. Raise the threshold the moment you see a hit that answers a different question.

## 3. Embedding reuse

```python
import hashlib
def embed_cached(texts: list[str], model: str) -> list[list[float]]:
    keys = [hashlib.sha256(f"{model}\n{t.strip()}".encode()).hexdigest() for t in texts]
    found = store.get_many(keys)                      # your KV / vector DB
    missing = [i for i, k in enumerate(keys) if k not in found]
    if missing:
        resp = client.embeddings.create(model=model, input=[texts[i] for i in missing])  # batched
        for i, d in zip(missing, resp.data):
            found[keys[i]] = d.embedding
        store.put_many({keys[i]: found[keys[i]] for i in missing})
    return [found[k] for k in keys]
```

```typescript
import { createHash } from "node:crypto";
export async function embedCached(texts: string[], model: string): Promise<number[][]> {
  const keys = texts.map(t => createHash("sha256").update(`${model}\n${t.trim()}`).digest("hex"));
  const found = await store.getMany(keys); // Map<string, number[]>
  const missing = keys.map((k, i) => (found.has(k) ? -1 : i)).filter(i => i >= 0);
  if (missing.length) {
    const resp = await client.embeddings.create({ model, input: missing.map(i => texts[i]) });
    missing.forEach((i, j) => found.set(keys[i], resp.data[j].embedding));
    await store.putMany(missing.map(i => [keys[i], found.get(keys[i])!]));
  }
  return keys.map(k => found.get(k)!);
}
```

Also: chunk deterministically (same chunker settings produce the same chunks and therefore the same hashes), re-embed only changed documents, and move bulk re-embedding to a Batch API.
