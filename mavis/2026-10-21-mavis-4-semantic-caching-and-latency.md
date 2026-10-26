---
layout: default
title: "Part 4: Semantic Caching"
parent: "MAVIS Series"
nav_order: 5
permalink: /posts/2026-10-21-mavis-4-semantic-caching-and-latency/
date: 2026-10-21 13:00:00 +0000
categories: [mavis, architecture]
summary: "How MAVIS cuts latency and token costs: SQLite vector caching, tiered similarity thresholds, separating plan reuse from result TTLs, and deterministic regex pre-filters."
---

# Part 4: Semantic Caching
{: .fs-8 }

How MAVIS cuts latency and token costs: SQLite vector caching, tiered similarity thresholds, separating plan reuse from result TTLs, and deterministic regex pre-filters.
{: .fs-5 .fw-300 .text-grey-dk-000 }

---

## The Fault in Our Caches

Caching is a core concept in software and hardware design where you store results for reuse instead of recomputing. MAVIS has to construct a DAG for every task and then execute it which is significant in terms of both latency and cost. Therefore, it is more optimal to store and re-use both plans and results whenever possible. However, there are three issues that must be solved:

- **When to Use Cache?**: Doing an exact match of query to determine a cache hit will result in misses where queries are reworded or are very similar. Depending upon semantic matching can result in serving wrong results for queries like "Summarize that" or "Summarize something that is similar to X". An LLM can figure that out but it is slow and costly for caching.
- **What to serve from Cache?**: Since results and pipelines both can be served from cache, it is important to determine whether the results are good to be reused or if the pipeline needs to be re-run. 
- **How long to serve?**: Cached results are prone to becoming stale. For example, if you cached the date for the query, "What is today's date?" and the user prompted for it again tomorrow, a wrong result is served. Therefore, determining an appropriate TTL is necessary.


MAVIS solves this with an SQLite-based cache, tiered cache hit handling, structural pre-filtering, and a strict architectural distinction between **plan reuse** and **result reuse**.

---

## Cache Pre-filtering: From Naive Regexes to Structural Filtering

As mentioned above, natural language queries such as "Fix it", "Could you repeat that?", etc. are notoriously difficult to cache and serve properly. Because their meaning depends entirely on previous conversational context, extracting the relevant context without an LLM is error-prone, and caching them pollutes the database with ambiguous keys. MAVIS avoids caching them altogether using a regex-based pre-filtering that identifies contextual queries by looking for the following signals:

- **Continuation Openers**: Queries starting with *also*, *now*, *then*, *keep*, *again*, *continue*.
- **Anaphoric Pronouns & Determiners**: References like *it*, *this*, *that*, *the same*, *the other*, *the previous*.
- **Conversational Feedback**: Agreements and transitions (*"that worked"*, *"sounds good"*, *"not quite"*).
- **Elliptic Commands**: Short flow controls (*"why?"*, *"continue"*, *"undo"*).

> There is one exception to these signals. Queries containing self-contained payloads (e.g., *"Explain this code: def foo()..."*) are still considered for caching as the context is provided in the query itself.

Queries identified as context-dependent bypass cache lookup and cache writing entirely.

> **Note**: Cache hit rates on the dashboard are calculated strictly over non-bypassed queries. The bypass rate is reported as a separate metric so statistics are never artificially skewed.

---

## Tiered Similarity Thresholds

When a query arrives after passing the cache pre-filter, it is embedded and compared against stored query vectors using cosine similarity. The similarity score is used to determine the cache hit strategy as given below: 
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
     └─► Score < 0.85 ──────────► Tier 3: Cache Miss ──► Full Interpreter Planning
```
This is done to ensure that on high or low similarity queries, the cache hits and misses are determined instantly and for slightly ambiguous cases, a slower but more capable LLM verifier can help determine if this should be a cache hit or miss. This helps maintain the benefits of caching while still considering correctness.
---

## Storing in Cache

Not every query or execution result is worth caching. To determine how an execution should be stored, the `Interpreter` produces three caching parameters alongside the compiled DAG `pipeline`:

1. **`cacheable` (Boolean)**: Dictates whether the query can be cached at all, or if it involves non-repeatable operations or transient time references.
2. **`ttl` (Seconds)**: Defines how long the computed result remains fresh before expiring.
3. **`generalizability` (`"generalized"` vs. `"specialized"`)**: Indicates whether the generated pipeline can serve as a reusable template for similar requests with different arguments, or if it is strictly tied to this specific query.

### How These Parameters Determine Storage

Before writing to the cache, MAVIS evaluates these parameters alongside the input:
* **Pre-Filter Rejection**: If `cacheable` is `False`, or if the query triggers the context pre-filtering, the execution bypasses storage entirely to prevent polluting the cache.
* **Recording Plan and State**: For eligible queries, MAVIS stores the query vector alongside the compiled DAG `pipeline`, the computed `result`, the expiration timestamp (`now + ttl`), and the `generalizability` flag.
* **Generalizability Tagging**: Entries marked `"generalized"` signal that future near-matching queries don't need to match identically; a targeted LLM parameter extraction pass (`_extract_parameters`) simply extracts arguments from the new query and slots them into the existing DAG template nodes. Entries marked `"specialized"` are reserved strictly for near-exact query reuse.

---

## Plan Reuse vs. Result Reuse: Why TTL Matters

A key insight in MAVIS is separating reusing the DAG pipeline from reusing the result directly; reusing results is valid for queries whose answers remain static over time (e.g., *"What is an AST?"* or *"Explain topological sort"*). These entries are assigned long TTLs and return instantly without running any code. On contrary, reusing plans is valid for dynamic queries whose answers change but whose execution structure remains identical (e.g., *"Check git status of repo X"* or *"What is the weather in city Y?"*). The pipeline is reused, using a targeted LLM parameter extraction pass to map new arguments into the cached DAG before re-execution.

### Storage, Cache Eviction, and Maintenance

To avoid external vector database dependencies, MAVIS stores cache embeddings in a standard SQLite table. At lookup time, candidate rows are evaluated using in-memory Python cosine similarity, keeping retrieval fast without requiring a dedicated vector DB.

Because plans and results have different lifecycles, cache eviction operates asymmetrically. When an entry's TTL expires, MAVIS does not delete the cached record—it only clears the `result` field while keeping the DAG `pipeline` intact. This allows stale data to be discarded without losing the pipeline. The next matching query gets a fast pipeline hit, running the existing plan to fetch fresh data without incurring the latency or cost of full planning. 

To prevent unbounded storage growth over time, the cache enforces a bounded ceiling (1,000 entries), pruning the oldest records when the limit is reached using LRU(Least Recently Used) cache eviction strategy.

---

*In Part 5, I will explain how MAVIS does autonomous self-reflection during idle periods without infinite token burn.*


---

<div style="display: flex; justify-content: space-between; align-items: center; margin-top: 2rem; padding-top: 1rem; border-top: 1px solid #eee; font-size: 0.9rem;">
  <div>← Previous<br><a href="{{ "/posts/2026-10-21-mavis-3-memory-architecture/" | relative_url }}"><strong>Part 3 — Memory Architecture</strong></a></div>
  <div style="text-align: right;">Next →<br><a href="{{ "/posts/2026-10-21-mavis-5-autonomous-self-reflection/" | relative_url }}"><strong>Part 5 — Autonomous Reflection</strong></a></div>
</div>
