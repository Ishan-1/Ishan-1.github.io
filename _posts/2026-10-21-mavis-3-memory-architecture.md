---
layout: post
title: "MAVIS Part 3 — Memory Architecture: 3 Tiers, Discrete Labels, and Topic-based Knowledge Graphs"
date: 2026-10-21 12:00:00 +0000
categories: [mavis, architecture]
series: mavis
series_part: 3
summary: "How MAVIS manages context across sessions without catastrophic forgetting: three cognitive tiers, verified promotion criteria, and topic-subscribed Neo4j knowledge graphs."
---

## To Remember or to Forget?
Using an AI agent that remembers nothing is like interacting with a person who forgets what you said a minute ago. On the flip side, trying to remember everything is a recipe for disaster: bloated context degrades model reasoning, pollutes prompts with conversational noise, and drives up token costs.

Therefore, dumping raw chat history into context memory is a sure way to blow up the entire context window with irrelevant content to the query. This can be handled by doing a semantic retrieval of the relevant memory from the vector database and injecting top-K similarity matches into the context window. But this simple fix breaks down quickly in practice. If you told the agent three months ago that you use VS Code, and yesterday instructed it that you switched to Neovim, similarity search retrieves both statements. The LLM is left to guess which one is authoritative. 

Even worse, unpartitioned memory violates basic security boundaries: why should an autonomous code-generation tool synthesizing a bash command have read access to your personal notes or calendar entries?

Therefore, the central question I considered when designing the memory was "What to Remember and What to Forget?".

To solve this, MAVIS models memory after human cognitive consolidation: three distinct retention tiers with explicit promotion gates, and topic-subscribed knowledge graphs.

---

## Three Tiers of Cognitive Retention

```
[ Working Memory ] (In-memory buffer, capped at min(MAX_TOKENS, 0.4 * Context))
        │
        ▼ (In-session compaction on overflow / 15-min background worker)
[ Short-Term Memory ] (Rolling 7-day daily JSON files + ChromaDB vector collection)
        │
        ▼ (8-hr background consolidation worker: directive & keyword gating)
[ Long-Term Memory ] (Topic-Subscribed Neo4j Knowledge Graph + Partitioned JSON)
```

| Tier | Lifetime | Storage Format | Promotion / Persistence Trigger |
|---|---|---|---|
| **Working Memory** | Active session | In-memory Python list | Session turn initialization |
| **Short-Term Memory** | 7 Days (rolling) | Daily JSON files (`YYYY-MM-DD.json`) + ChromaDB | 4 triggers (emotion, directive, $\ge 3\times$ repetition, tool failure) via 15-min worker |
| **Long-Term Memory** | Indefinite | Neo4j property graph + partitioned JSON (`behaviours`, `facts`, `patterns`, `fixes`) | Directive + keyword gating via 8-hr worker; direct pattern/fix writes from ToolBuilder & Debugger |

---

### Working Memory → Short-Term Promotion (15-Minute Worker)

Working memory holds raw turns in an in-memory buffer. When the buffer approaches token capacity, an in-memory compaction step summarizes older conversational exchanges.

In parallel, a background worker (`short_term_worker.py`) runs every 15 minutes during active sessions (guarded to skip if the session has been idle for $>30$ minutes). It inspects unprocessed turns and promotes an entry to short-term daily JSON storage and ChromaDB if any of four triggers fire:

1. **Emotion Strength**: Detected user urgency, frustration, or excitement (`emotion_strength == "high"` or $\ge 0.75$).
2. **Explicit Directive**: The turn contains a clear preference or rule (`directive == True`).
3. **Repetition**: A turn's embedding matches previous turns with cosine similarity $\ge 0.85$ at least 3 times within a 50-turn window.
4. **Tool Failure**: The entry records a runtime tool fault (`tool_failure == True`), preserving context for subsequent debugging.

---

### Short-Term → Long-Term Promotion (8-Hour Worker & Subsystems)

Promoting something to indefinite long-term storage requires much higher confidence. In MAVIS, this is split into two mechanisms:

#### 1. The Periodic Consolidation Worker (`long_term_worker.py`)
Every 8 hours (480 minutes via the internal task runner), a worker scans short-term turns added since its last persisted cursor (`lt_cursor_path`). Unlike short-term promotion, emotion and repetition alone **cannot** promote an entry to long-term memory.

The worker evaluates entries through `should_promote_long_term()`:
- **Mandatory Hard Gate**: The turn must have `directive == True` (or legacy `intent_strength > 0.85`). If `directive` is false, promotion is rejected immediately.
- **Persistent Behavioral Keywords**: If the content contains terms like `"always"`, `"never"`, `"permanently"`, `"from now on"`, `"every time"`, or `"prefer"`, it is promoted as a **`behaviour`** (`behaviours.json`).
- **Explicit Fact Keywords**: If the content contains terms like `"remember forever"`, `"remember this permanently"`, `"never forget"`, or `"remember that"`, it is promoted as a **`fact`** (`facts.json`).
- **Knowledge Graph Extraction**: When an entry qualifies, an asynchronous extractor (`extract_facts_from_turn`) runs an LLM pass to convert the turn into structured `(subject, predicate, object)` triples and writes them into Neo4j.
- **7-Day TTL Pruning**: After processing delta turns, the worker deletes short-term daily JSON files older than 7 days and purges expired date ranges from the ChromaDB collection.

#### 2. Direct Subsystem Writes (No Background Worker Wait)
Specialized autonomous subsystems bypass the 8-hour worker entirely when recording verified technical knowledge:
- **`ToolBuilder`**: Once a synthesized tool passes static analysis and sandbox tests, it immediately calls `write_pattern()`, persisting the function signature and conventions into `patterns.json` and creating a tool definition node in Neo4j.
- **`PipelineDebugger` & `AgentBuilder`**: When a debugging loop successfully fixes a tool execution fault or agent prompting failure, it immediately calls `write_fix()` / `write_agent_fix()`, logging the failure mode and verified patch into `fixes.json` and adding fix relationships to Neo4j.

---

## Why Discrete Labels Beat Continuous Floats

Many agent architectures ask LLMs to emit continuous floating-point scores between `0.0` and `1.0` for sentiment, importance, or intent strength.

In practice, LLM-generated floats are noisy, uncalibrated pseudo-math. A score of `0.78` vs `0.84` does not reflect a meaningful probability distribution—it reflects prompt phrasing quirks and sampling noise.

Instead of parsing fragile floats, MAVIS classifies entries using **discrete categories** and boolean flags:
- **Emotion**: `"low"`, `"medium"`, `"high"`
- **Directive**: `True` / `False`

These fields are extracted directly in the primary `Interpreter` response schema. Because they are returned as part of the normal turn comprehension pass, classification incurs zero additional API round-trips or latency overhead.

> **Engineering Principle**: Reject continuous LLM-emitted confidence floats in favor of discrete enum buckets and binary flags that map directly to deterministically testable state transitions.

---

## Topic-based Knowledge Graphs

Different components in MAVIS require fundamentally different kinds of memory:
- The **Interpreter** needs user preferences, environment configs, and episodic history to understand user requests.
- The **ToolBuilder** and **PipelineDebugger** need OS specifications, Python runtime details, and prior fix patterns.
- **Background Tasks** need execution state and workflow metadata.

Mixing all of this into a single global vector store creates context cross-contamination and introduces major security risks. Interacting with memory is therefore modeled as a **publisher-subscriber hierarchy** over partitioned Neo4j graphs and JSON stores:

| Component Namespace | Subscribed Read Topics | Write Topic | User Personal Data Access |
|---|---|---|---|
| **Interpreter** | `["user.*", "env.*", "tooling.*", "tasks.*"]` | `user.episodic` | ✅ Permitted |
| **ToolBuilder / Debugger** | `["env.os", "env.python", "tooling.*"]` | `tooling.patterns` | ❌ **Hard Blocked** |
| **Background Tasks** | `["env.*", "tooling.*", "tasks.*"]` | `tasks.workflow` | ❌ **Hard Blocked** |

Code-generating components like `ToolBuilder` are **strictly barred** from querying personal user namespaces (`user.*`). Even if a prompt injection exploits the tool synthesis pipeline, it cannot read personal memory or credentials.

### Resolving Contradictions: Superseding, Not Deleting

User preferences and environment states evolve over time. When a new 1-to-1 functional predicate arrives (e.g., updating `PREFERS_EDITOR = "Neovim"` where `PREFERS_EDITOR = "VS Code"` previously existed), the previous edge is not erased. 

Instead, the graph engine marks the prior relationship edge as inactive with an expiration timestamp. The historical record is preserved for auditing and rollbacks, but subscriber queries filter for active edges only. This prevents vector search's classic failure mode: surfacing two contradictory facts with equal weight.

---

*In Part 4, I will discuss how MAVIS implements semantic caching to cut latency and cost on redundant queries.*
