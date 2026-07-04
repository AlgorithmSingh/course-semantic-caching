# Build Your First Semantic Cache

Build a semantic cache from scratch, then redo it with Redis + RedisVL.
Use case: customer support FAQ.

## Recap flow

![Recap semantic caching flow](recap-semantic-caching-flow.png)

```text
Query -> Embeddings Service -> Semantic Cache
   cache hit  -> return answer straight to user
   cache miss -> RAG (Vector Database -> LLM) -> Response -> user
                 then Update Cache with the new Q/A
Sourced FAQs preload the cache up front.
```

Caveman: check meaning first. Seen it? answer fast. Not seen? do RAG, then remember it.

## Part 1 — from scratch (numpy)

Goal: see every moving part before hiding it behind an SDK.

Steps:

1. **Load FAQ dataset** — question/answer pairs + test set (from CSV).
2. **Embed** with `SentenceTransformer("all-mpnet-base-v2")` — encode all FAQ questions into vectors.
3. **Semantic search** — cosine distance between query embedding and the FAQ matrix; pick lowest distance (`argmin`).
4. **check_cache(query, distance_threshold=0.3)** — if best distance ≤ threshold → HIT (return prompt+response+distance); else → MISS (None).
5. **add_to_cache(q, a)** — append row to the dataframe and `vstack` the new embedding onto the matrix. Cache grows over time.

Key numbers from the lab:

- "How long will it take to get a refund for my order?" → nearest FAQ "How do I get a refund?", distance `0.331`.
- "Is it possible to get a refund?" → HIT at `0.262`.
- After adding 3 entries (8 → 11), previously-missing queries ("What time do you open?", etc.) now HIT.

Mental model:

```text
distance = 1 - cosine_similarity
lower distance = closer meaning
threshold = how strict:  loose = more hits + more risk,  strict = fewer hits + safer
```

## Part 2 — production with Redis

**Redis** = REmote DIctionary Server. Open-source, fast, in-memory key-value DB.
It also does secondary indexing (vectors, text, numerics, tags, geo) — that is what makes
semantic caching in Redis possible.

Why Redis over the numpy version:

- Low-latency vector search (fast insert + check at scale).
- Distributes data across nodes; apps read/write concurrently.
- Built-in **TTL / eviction / namespacing** for freshness and tenant isolation.

Pipeline in code:

```python
r = redis.Redis.from_url(REDIS_URL)         # redis://localhost:6379

# cache-optimized, fine-tuned embedding model (open weight, HuggingFace)
langcache_embed = HFTextVectorizer(
    model="redis/langcache-embed-v1",
    cache=EmbeddingsCache(redis_client=r, ttl=3600),
)

cache = SemanticCache(
    name="faq-cache",           # unique namespace in the DB
    vectorizer=langcache_embed,
    redis_client=r,
    distance_threshold=0.3,
)

for i in range(len(faq_df)):    # hydrate cache with FAQ data
    cache.store(prompt=faq_df.iloc[i]["question"],
                response=faq_df.iloc[i]["answer"])

cache.check("I need a refund for my purchase")   # -> hit at distance 0.25 (< 0.3)
cache.set_ttl(86400)            # keep cache fresh: evict after 1 day
```

Notes:

- `langcache-embed-v1` is fine-tuned specifically for caching → better hit accuracy than the generic mpnet model.
- `name=` creates a namespace; different tenants/apps stay isolated.
- `EmbeddingsCache(ttl=3600)` also caches the *embeddings* themselves (don't re-embed the same text).

## Part 3 — end-to-end with LLM + perf eval

Loop: for each test question → `cache.check` → HIT return cached; MISS call `gpt-4o-mini`
(via LangChain `ChatOpenAI`), then `cache.store` the fresh answer.

Result:

- Cache hit ≈ **65 ms** average.
- LLM call ≈ **> 1 second** average.
- (LLM latency here is mocked/random, but the gap is representative.)

Takeaway: cache turns a ~1s+ LLM call into a ~65ms lookup for repeated meaning.
Next lesson covers measuring cache effectiveness (accuracy + performance metrics) properly.

## Redis primer (from side chats)

Got confused on Redis basics — here is the grounded version.

**Key vs value**: the key is the name/label you choose; the value is the data stored under it.
`SET name Alice` → key `name`, value `Alice`. Redis does **not** auto-generate keys; you name them
(e.g. `user:123`, `orders`, `faq-cache`).

**Data types** = the *shape* of the value. Same key→value idea, different container:

- String — one value.
- Hash — labeled fields (an object), commands start `H` (HSET/HGET).
- List — ordered queue, `L` (LPUSH/LPOP).
- Set — unique items, `S` (SADD).
- Sorted set — `Z` (ZADD).
- Stream — append-only event log, `X` (XADD/XREAD).

**Command naming trick**: the first letter tells you the data type; the rest is a plain verb.
`XADD` = X (stream) + ADD. Not a fancy acronym. `HGETALL` = hash + get + all. `LPUSH` = list + push.
(X and Z were picked mostly because the good letters were already taken.)

**Streams specifics** (for the Redis Stream data type):

```text
XADD orders * user_id 123 item "book" price 25
      |     |  \_______ field/value pairs ______/
      key   auto-generate entry ID (*)

orders  -> Stream
  1692632147973-0 : user_id=123, item=book, price=25   <- entry ID = timestamp-counter
```

- **entry ID** like `1692632147973-0` = timestamp + a counter (for same-millisecond collisions); like a line number in a logbook.
- `*` = "Redis, generate the entry ID for me."
- **field/value pairs** = labeled details inside one entry, so the event is self-describing.

Why commands exist at all: Redis is a server holding data in memory; commands are the only
language to tell it store/fetch/append. No command = silent box you can't reach.

## Caveman summary

- Build cache from scratch = embed FAQs, cosine distance, threshold 0.3, grow over time.
- Redis version = same idea but fast, scalable, with TTL + namespaces.
- Fine-tuned `langcache-embed-v1` beats generic embeddings for cache hits.
- Hit ≈ 65ms vs LLM ≈ 1s+ → big win on repeats.
- Redis basics: key = name, value = data, data type = shape, command prefix = data type + verb.
