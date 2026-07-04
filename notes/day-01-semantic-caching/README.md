# Semantic Caching Notes — 2026-07-04

## Course idea

Semantic cache = cache by meaning, not exact words.

Example:

- “How can I get a refund?”
- “I want my money back.”

Exact cache sees different words. No hit.
Semantic cache sees close meaning. Possible hit.

## How it works

- Make embedding for new question.
- Compare with old cached questions.
- If distance/similarity passes threshold, reuse old answer.
- If not, call model and store new answer.

Caveman version:

- Exact cache: same words, use old answer.
- Semantic cache: same meaning, maybe use old answer.

## Main danger

Same meaning does not always mean same answer.

Example:

- User A: “What is refund policy for my product?”
- User B: “What is refund policy for my product?”

Words same. Intent same. But context different.

- User A bought phone. Refund window maybe 7 days.
- User B bought TV. Refund window maybe 15 days.

If cache only sees text, it may return wrong policy.

Caveman rule:

> Same words not enough. Same meaning not enough. Need same context.

## Better design

Use context-aware semantic caching.

Cache should include things like:

- intent: refund policy
- user or tenant
- order id
- product type
- product category
- region/store
- purchase date
- policy version/date

Do not blindly cache:

```text
"refund policy" -> "15 days"
```

Better cache inside context bucket:

```text
intent=refund_policy
product_type=TV
region=US
policy_version=2026-07
answer="TV can be returned within 15 days."
```

## Safe flow

1. Understand user intent.
2. Fetch user/order/product facts from core database.
3. Apply correct policy.
4. Check semantic cache only inside same context.
5. If cache hit is safe, reuse answer.
6. If not safe, call LLM/tool and store answer with metadata.

## Threshold tradeoff

Threshold controls how strict match is.

- Loose threshold = more hits, more risk.
- Strict threshold = fewer hits, safer answers.

Metrics from course:

- Hit rate: how often cache helps.
- Precision: when cache says hit, how often correct.
- Recall: how many correct possible hits it catches.

## Ways to improve

Course mentions:

- Redis semantic cache SDK.
- TTL so stale answers expire.
- Separate caches per user/team/tenant.
- Cross-encoder re-ranking for better match check.
- Small LLM check: “Are these questions really same?”
- Fuzzy matching for typos.
- Confusion matrix to see wrong hits/misses.

## Walmart / waLLMartCache example

![Walmart accuracy table](walmart-accuracy-table.png)

Course transcript + slide mention Walmart-style production semantic cache: `waLLMartCache`.

Paper (now confirmed): waLLMartCache — *A Distributed, Multi-tenant and Enhanced Semantic Caching System for LLMs*.
Link: https://link.springer.com/chapter/10.1007/978-3-031-78183-4_15

Accuracy table read from slide:

| Method | Acc Reg | Acc All |
|---|---|---|
| Oracle | 100 | 100 |
| GPTCache | 86.4 | 64.8 |
| WMC(1N) | 86.4 | 64.8 |
| WMC(4N) | 86.1 | 64.5 |
| WMC(4N)+DE | 86.1 | 80.2 |
| **WMC(4N)+DE+FAQ** | **89.6** | **81.4** |

Big jump comes from `+DE` (Decision Engine) and `+FAQ` preload, not from more nodes.
Adding nodes alone (`1N` → `4N`) barely moves accuracy — it is about scale, not correctness.

Big point:

> Semantic cache alone is not enough for production.

Walmart-style system adds extra safety:

- Redis/distributed cache: cache works across many nodes.
- Decision Engine: decides when cache should NOT answer.
- FAQ preload: trusted common answers are loaded before users ask.
- Best row in slide: `WMC(4N)+DE+FAQ`.
- Accuracy shown: `89.6%` for regular queries, `81.4%` overall/all.

Decision Engine can bypass cache for risky queries:

- code questions
- time-sensitive questions
- questions needing fresh data
- questions needing user/product/order context

If risky, go normal LLM/RAG/database path.

Caveman version:

- Cache is shortcut.
- Shortcut not always safe.
- Decision Engine asks: “safe to use shortcut?”
- If no, do real lookup.

The best setup in transcript was WMC + Decision Engine + FAQ preload.
It got close to 90% accuracy because wrong cache hits reduced.

This supports our doubt:

Refund question may look same, but product/user/time can change answer.
So production cache needs guardrails, metadata, and database grounding.

Better name for this full setup:

- production semantic caching with decision engine
- context-aware semantic caching
- semantic cache with policy/metadata guardrails

## Distributed architecture (L1 / L2)

![Walmart distributed caching architecture](walmart-distributed-architecture.png)

waLLMartCache is a distributed caching service with load balancing across many nodes.

Flow:

- Users send queries to a **load balancer**.
- Load balancer routes to one of several `waLLMartCache` nodes.
- Each node has a **Distributed Cache Manager** with two tiers.

Dual-tiered storage:

- **L1 = Vector Database** (retrieval). Holds embeddings, does similarity search.
- **L2 = In-Memory Cache / Redis** (lookup). Holds the actual saved answer by a scalar id.

So flow inside a node:

```text
query -> L1 vector search (find similar question) -> scalar id -> L2 lookup (get saved answer)
```

**Multi-tenancy**: L1 and L2 storage is partitioned per tenant (Tenant 1..4).
Each team/app gets its own bucket so one tenant's cache never leaks to another.

Caveman version:

- Many nodes. Load balancer picks one.
- L1 = find match by meaning.
- L2 = fetch the real answer fast.
- Each tenant has own drawer.

## Decision Engine flow (the gate)

![Walmart Decision Engine flow](walmart-decision-engine.png)

Decision Engine improves cache **precision** beyond plain semantic search.
It is the "should we even use cache?" gate, checked before similarity search.

Flow:

```text
query
  -> Code Detector?        yes -> LLM (skip cache)
       no
  -> Temporal Context?     yes -> LLM (skip cache)
       no
  -> Similarity Evaluator  -> cache hit  -> return cached answer
                           -> cache miss -> LLM, then store answer
```

Why each gate:

- **Code Detector**: code queries are risky, tiny diff changes the answer. Don't cache.
- **Temporal Context Detector**: time-sensitive queries (price today, latest status, current policy) go stale. Don't cache.
- **Similarity Evaluator**: only safe, normal questions reach here — the actual semantic lookup.

Caveman version:

- Code? don't cache.
- Time-sensitive? don't cache.
- Safe normal question? try semantic cache.
- Hit = fast answer. Miss = ask LLM.

## Three cache types (grounded)

From RedisVL docs + RAG-caching papers, these are different things people call "semantic cache":

- **Semantic response cache**: stores the final LLM answer. Key = prompt (+ filters), value = answer + metadata. Supports threshold, TTL, access control. (RedisVL `SemanticCache`)
- **Retrieval / RAG cache**: stores retrieved docs/DB results *before* generation. On hit, skip the vector DB lookup; LLM still writes a fresh answer. (arXiv 2503.05530)
- **Profile / context filters**: `user_id`, `tenant`, `account_type`, `region` on each entry — prevents wrong cross-user reuse.

One-line rule:

```text
Do not cache just: meaning(query)
Cache with:        meaning(query) + filters/context
Bypass cache when: code, time-sensitive, risky, or context missing
```

The refund example is exactly **context-aware semantic caching** — plain query-only cache gives wrong hits when similar queries have different hidden context (arXiv 2506.22791 ContextCache).

Grounding sources:

- RedisVL LLM cache: https://redis.io/docs/latest/develop/ai/redisvl/user_guide/llmcache/
- Approximate caching for RAG: https://arxiv.org/abs/2503.05530
- ContextCache (multi-turn): https://arxiv.org/abs/2506.22791
- waLLMartCache paper: https://link.springer.com/chapter/10.1007/978-3-031-78183-4_15

## Course project — what we build

![Course project: deep research agent with semantic caching](course-project-deep-research-agent.png)

End goal: a **LangGraph** deep-research agent with semantic cache inside.

Graph flow:

```text
start -> decompose_query -> check_cache -> research -> evaluate_quality -> synthesize -> end
```

- **decompose_query**: split one big question into smaller sub-questions.
- **check_cache**: for each sub-question, look in semantic cache first.
- **research**: on miss, scrape/search loaded content + call LLM.
- **evaluate_quality**: not good enough? loop back and research more.
- **synthesize**: stitch all sub-answers into one personalized final answer.

Why cache is highlighted: one agent run fires many sub-questions. Without cache every
sub-question re-hits the LLM/RAG. With cache, repeated sub-questions get cheap.

Demo UI: load a website URL → extract chunks → build knowledge base → ask question →
performance log shows per-step LLM/cache hits, time, and tokens.

## Is this hybrid search?

Not exactly.

Hybrid search usually means:

```text
keyword search + vector search
```

This case is better called:

- context-aware semantic caching
- semantic cache with metadata filters
- database-grounded semantic cache
- RAG-aware caching

Caveman difference:

- Hybrid search: find using words plus meaning.
- Context-aware cache: same meaning plus same facts/context.

## Final caveman summary

Semantic cache good. Saves cost and time.

But if answer depends on user/product/order, cache must know that.

Meaning match alone can lie.

Need meaning + context + freshness check.
