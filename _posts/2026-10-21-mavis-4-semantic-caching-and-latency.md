---
layout: post
title: "MAVIS Part 4 — Semantic Caching: Plan vs. Result Reuse"
date: 2026-10-21 13:00:00 +0000
categories: [mavis, architecture]
series: mavis
series_part: 4
summary: "How MAVIS cuts latency and token costs: SQLite vector caching, tiered similarity thresholds, separating plan reuse from result TTLs, and deterministic regex pre-filters."
---

## The Fault in Our Caches

Caching is a core concept in software and hardware design where you store results for reuse instead of recomputing. MAVIS has to construct a DAG for every task and then execute it which is significant in terms of both latency and cost. Therefore, it is more optimal to store and re-use both plans and results whenever possible. However, there are three issues that must be solved:

- **When to Use Cache?**: Doing an exact match of query to determine a cache hit will result in misses where queries are reworded or are very similar. Depending upon semantic matching can result in serving wrong results for queries like "Summarize that" or "Summarize something that is similar to X". An LLM can figure that out but it is slow and costly for caching.
- **What to serve from Cache?**: Since results and pipelines both can be served from cache, it is important to determine whether the results are good to be reused or if the pipeline needs to be re-run. 
- **How long to serve?**: Cached results are prone to becoming stale. For example, if you cached the date for the query, "What is today's date?" and the user prompted for it again tomorrow, a wrong result is served. Therefore, determining an appropriate TTL is necessary.


MAVIS solves this with a lightweight **SQLite vector cache**, tiered cache hit thresholds, cache pre-filtering, and a strict architectural distinction between **plan reuse** and **result reuse**.


---

## Cache Pre-filtering

As mentioned above, natural language queries such as "Fix it", "Could you repeat that?", etc. are notoriously difficult to cache and serve properly. Because their meaning depends entirely on previous conversational context, extracting the relevant context without an LLM is error-prone, and caching them pollutes the database with ambiguous keys. MAVIS avoids caching them altogether using a deterministic regex pre-filter:

```python
# Context-dependent queries bypass cache lookup and writes completely
CONTEXT_DEPENDENT_PATTERNS = [
    r"^(fix|repair|patch)\s+(this|that|it)",
    r"^(continue|keep going|next)",
    r"^(try again|retry)",
    r"^why did (that|it) fail"
]
```

Queries matching these patterns skip the cache entirely to avoid polluting the database with context-dependent keys. 

> **Note**: Cache hit rates on the dashboard are calculated strictly over **eligible (non-bypassed)** queries. The bypass rate is reported as a separate metric so statistics are never artificially skewed.

---

## Tiered Similarity Thresholds

When a query arrives after passing the cache pre-filter, it is embedded and compared against stored query vectors using cosine similarity:
```
Incoming Query
     │
     ▼
[ SQLite Vector Search ] ──► Computes Cosine Similarity
     │
     ├─► Score > 0.95 ──────────► Tier 1: Instant Hit
     │                             └─► Valid TTL? ──► Return cached result
     │                             └─► Stale TTL? ──► Re-execute cached DAG
     │
     ├─► 0.85 ≤ Score ≤ 0.95 ───► Tier 2: Fast LLM Verifier Check
     │                             └─► Confirmed? ──► Reuse pipeline
     │                             └─► Rejected?  ──► Pass to Interpreter
     │
     └─► Score < 0.85 ──────────► Tier 3: Automatic Miss ──► Full Interpreter Planning
```

---

## Storing in Cache

Not every query or execution result is worth caching. To determine how an execution should be stored, the `Interpreter` produces three caching parameters alongside the compiled DAG `pipeline`:

1. **`cacheable` (Boolean)**: Dictates whether the query can be cached at all, or if it involves non-repeatable operations or transient time references.
2. **`ttl` (Seconds)**: Defines how long the computed result remains fresh before expiring.
3. **`generalizability` (`"generalized"` vs. `"specialized"`)**: Indicates whether the generated pipeline can serve as a reusable template for similar requests with different arguments, or if it is strictly tied to this specific query.

### How These Parameters Determine Storage

Before writing to the cache, MAVIS evaluates these parameters alongside the input:
* **Pre-Filter Rejection**: If `cacheable` is `False`, or if the query triggers the regex context-dependency pre-filter (e.g., *"try that again"*), the execution bypasses storage entirely to prevent polluting the cache.
* **Recording Plan and State**: For eligible queries, MAVIS stores the query vector alongside the compiled DAG `pipeline`, the computed `result`, the expiration timestamp (`now + ttl`), and the `generalizability` flag.
* **Template Tagging**: Entries marked `"generalized"` signal that future near-matching queries don't need to match identically; a lightweight parameter extraction pass can simply slot new variables into the existing DAG nodes. Entries marked `"specialized"` are reserved strictly for near-exact query reuse.

---

## Plan Reuse vs. Result Reuse: Why TTL Matters

A key insight in MAVIS is separating **how a task is accomplished** (the DAG pipeline) from **the output produced** (the result):

* **Result Reuse**: Valid for queries whose answers remain static over time (e.g., *"What is an AST?"* or *"Explain topological sort"*). These entries are assigned long TTLs and return instantly without running any code.
* **Plan Reuse**: Valid for dynamic queries whose answers change but whose execution structure remains identical (e.g., *"Check git status of repo X"* or *"What is the weather in city Y?"*). The pipeline is reused, but a lightweight parameter extractor maps new inputs into the cached DAG before re-execution.

### Cache Eviction and Maintenance
Because plans and results have different lifecycles, cache eviction operates asymmetrically. When an entry's TTL expires, MAVIS does not delete the cached record—it only clears the `result` field while keeping the DAG `pipeline` intact. This allows stale data to be discarded without losing the pipeline. The next matching query gets a fast pipeline hit, running the existing plan to fetch fresh data without incurring the latency or token cost of full planning. To prevent unbounded storage growth over time, the cache enforces a bounded ceiling (e.g., 1,000 entries), pruning the oldest records when the limit is reached.

---

*In Part 5, I will explain how MAVIS does autonomous self-reflection during idle periods without infinite token burn.*
