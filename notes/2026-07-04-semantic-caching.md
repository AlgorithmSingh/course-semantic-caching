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
