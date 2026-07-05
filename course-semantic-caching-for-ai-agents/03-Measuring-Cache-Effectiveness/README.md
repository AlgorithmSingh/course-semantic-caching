# Measuring Cache Effectiveness

How to evaluate a semantic cache: quality metrics (hit rate, precision, recall, F1),
latency/savings, and LLM-as-a-Judge for automatic labeling.

Caveman: cache can fail two ways — bad answers (quality) or slow/no savings (performance).
Measure both, like you measure an ML model.

## The eval dataset

Each row is one user query plus:

```text
query          -> the incoming user question
cache hit      -> closest cache entry we matched to
distance       -> how far the match was (lower = closer meaning)
label          -> was this match actually correct? (true / false)
```

The `label` column is the ground truth. You get it from:

- **Human feedback**, or
- **LLM-as-a-Judge** (an LLM decides if query and match really mean the same).

In the lab the dataset is auto-generated, so the correct labels are already known —
that is why `data_container.label_cache_hits(...)` can label results for free.

## Quality metrics

Picture the embedding space: each cache entry is a point with a **distance threshold**
circle around it. A query "hits" only if it lands inside that circle.

### Hit rate

```text
hit rate = queries inside the threshold / all queries
```

Of all traffic, how much the cache actually answers. Example: 3 of 5 → 60%.

### Precision

```text
precision = valid hits / all hits
```

Of the queries that *did* hit, how many were **correct** (label = true).
A hit that landed in the circle but is actually the wrong answer hurts precision.

### Recall

```text
recall = correct hits / queries that should have hit
```

Of the queries that *should* have matched, how many actually did. A query that
deserved a hit but fell outside the threshold (too strict) hurts recall.

### Confusion matrix

Every query lands in one of four buckets:

```text
                 predicted HIT        predicted MISS
should HIT     True Positive (TP)   False Negative (FN)
should MISS    False Positive (FP)  True Negative (TN)
```

- **TP** — matched and correct. Good.
- **TN** — no match and there shouldn't be. Good.
- **FP** — matched but wrong. Dangerous (serves a wrong cached answer).
- **FN** — should have matched but threshold was too tight. Missed savings.

Toy example from the slide: 2 TP, 1 TN, 1 FP, 1 FN. You want the mass on the
**main diagonal** (TP + TN).

### Threshold, and the precision/recall tradeoff

The distance threshold is the knob:

```text
lower threshold  -> stricter -> higher precision, lower recall
higher threshold -> looser   -> higher recall, lower precision
```

**F1 score** balances the two:

```text
F1 = 2 * (precision * recall) / (precision + recall)
```

Technique: **sweep the threshold** across a range and pick the one with the best F1.
(Next lesson uses this to find the "perfect" threshold.)

## Latency + savings

Not just quality — measure the speedup and the tokens saved.

### With-cache latency formula

```text
WCL = ALL * (1 - CHR) + ACL * CHR
```

- **ACL** — Average Cache Latency (how fast a cache hit responds).
- **ALL** — Average LLM Latency (how long the LLM takes).
- **CHR** — Cache Hit Ratio (fraction of queries that hit).

Slide example:

```text
ACL = 11 ms,  ALL = 350 ms,  CHR = 30%
WCL = 350*0.7 + 11*0.3 = 256 ms   -> ~26% latency improvement
```

### Token savings

Use the course's savings calculator URL: plug in cache hit rate, expected daily
queries, and input/output token counts → it estimates annual savings.

## In the notebook

### Wrapper + evaluator abstractions

```python
cache_wrapper = SemanticCacheWrapper.from_config(config)
cache_wrapper.hydrate_from_df(faq_df)          # preload FAQ
cache_wrapper.check(query)                      # single lookup -> matches object
cache_results = cache_wrapper.check_many(test_queries)   # batch; configurable threshold + #matches
```

```python
evaluator = CacheEvaluator(
    true_labels=data_container.label_cache_hits(cache_results),  # auto labels
    cache_results=cache_results,
)
evaluator.report_metrics()      # prints confusion matrix + metrics
```

Lab result at the default threshold: **precision 0.79, recall 0.79**.

### Confusion mask (not just numbers)

`get_metrics()` returns the usual numbers **plus** a `confusion_mask` — a boolean mask
over the dataset so you can pull out the actual rows in each bucket:

```python
[[tn, fp], [fn, tp]] = evaluator.get_metrics()["confusion_mask"]
evaluator.matches_df()[fp]      # inspect the false positives themselves
```

Why this matters: you can *read* the mistakes. Example FPs from the lab —

```text
"Can I get a refund if I change my mind?"  -> "How do I get a refund?"     (labeled false)
"Can I schedule a specific delivery time?" -> "Can I change my delivery address?"  (labeled false)
```

They *sound* similar, so the model gave a low distance, but the dataset labels them as
wrong matches → false positives. Tip: swap the mask (`fp` → `fn`, `tp`, `tn`) to explore
every group.

### Latency eval with PerfEval

```python
def simulate_llm_call(prompt):
    time.sleep(np.random.uniform(0.2, 0.5))   # mock LLM latency
    return f"LLM response to {prompt}"

with perf_eval:
    for query in tqdm(test_queries):
        cache_wrapper.check(query); perf_eval.tick("cache_hit")
        perf_eval.start(); simulate_llm_call(query); perf_eval.tick("llm_call")

metrics = perf_eval.get_metrics(labels=["cache_hit", "llm_call"])
perf_eval.plot(title="Performance Comparison", show_cost_analysis=False)
```

Lab numbers: cache ≈ **2.2 ms**, mocked LLM ≈ **361 ms** → raw **~161x speedup**.

**But raw speedup is not a fair number.** Not every query hits. Apply the real formula
with a realistic hit rate:

```python
cache_hit_rate = 0.3
cached_llm_latency = llm_latency*(1 - cache_hit_rate) + cache_latency*cache_hit_rate
# -> ~29% latency drop, ~1.42x overall app speedup
```

So: 161x is the best-case-per-hit; ~1.4x is the honest whole-system number at 30% hit rate.

## LLM-as-a-Judge (automatic labeling)

When you don't have human labels, let an LLM label the query/match pairs.

```python
# 1. Full retrieval: threshold=1 so EVERY query gets its nearest neighbor (even bad ones)
full = cache_wrapper.check_many(test_queries, distance_threshold=1)
matches = [h.matches[0].prompt for h in full]

# 2. Judge each (query, match) pair
evaluator = LLMEvaluator.construct_with_gpt()          # pre-built comparison prompt
results = evaluator.predict(dataset=zip(test_queries, matches), batch_size=5)
results.df    # -> columns: pair, reason, is_similar (bool + why)

# 3. Feed the LLM labels back in — note the special constructor for full retrieval
evaluator = CacheEvaluator.from_full_retrieval(
    true_labels=results.df["is_similar"].values,
    cache_results=cache_wrapper.check_many(test_queries),
)
evaluator.report_metrics()
```

Why `distance_threshold=1`: to judge true negatives you must first *see* the nearest
neighbor for every query, including the ones you'd normally reject.

Result vs ground truth:

```text
human/auto labels : precision 0.79, recall 0.79
LLM-as-a-Judge    : precision 0.75, recall 0.90
```

Close enough → LLM judging produces "good enough" labels when you have no human ones.

Cleanup: `cache_wrapper.cache.clear()`.

## Terms

- **Hit rate / CHR**: fraction of queries the cache answers.
- **Precision**: of hits, how many were correct.
- **Recall**: of deserved hits, how many happened.
- **F1**: harmonic balance of precision and recall; sweep threshold to maximize it.
- **Confusion mask**: boolean mask to pull the actual rows behind TP/TN/FP/FN.
- **WCL / ACL / ALL**: with-cache, avg-cache, avg-LLM latency.
- **LLM-as-a-Judge**: use an LLM to auto-label query/match correctness.

## Caveman summary

- Cache fails two ways: wrong answers (quality) or no speedup (performance). Measure both.
- Quality = precision (hits that are right) + recall (right hits you caught) → F1.
- Threshold is the knob: tighter = precise but low recall; looser = high recall but wrong hits.
- False Positive = matched but wrong = the scary one. Read them with the confusion mask.
- Latency: raw per-hit speedup (161x) is not honest; real app speedup at 30% hits ≈ 1.4x.
- No human labels? LLM-as-a-Judge labels almost as well (0.75/0.90 vs 0.79/0.79).
- Next lesson: sweep threshold with F1 to pick the best one, then improve the cache.
