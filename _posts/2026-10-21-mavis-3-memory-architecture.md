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
Using an AI agent that remembers nothing is like interacting with a person who forgets what you said a minute ago. On the flip side, trying to remember everything is a recipe for disaster as dumping raw chat history into context memory is a sure way to blow up the entire context window with irrelevant content to the query.

One way to handle this is by doing a semantic retrieval of the relevant memory from the vector database and injecting top-K similarity matches into the context window. But this has its own issues. If you told the agent three months ago that you use VS Code, and yesterday told it that you switched to Neovim, similarity search retrieves both statements when retrieving your preference and the LLM has to decide which one is correct.

Even worse, unpartitioned memory violates basic security boundaries: why should an autonomous code-generation tool synthesizing a bash command have read access to your personal notes or calendar entries?

Therefore, the central question I considered when designing the memory was "What to Remember and What to Forget?".

To solve this, MAVIS models memory after human memory: three distinct retention tiers with explicit promotion criteria, and topic-subscribed knowledge graphs for long-term memory.

---

## Three Tiers of Cognitive Retention
Analogous to human memory, MAVIS has the following kinds of memory with varying retention periods and promotion triggers to store different kinds of data:

| Tier | Purpose |  Lifetime | Storage Format | Promotion / Persistence Trigger |
|---|---|---|---|---|
| **Working Memory** | Store current conversational context| Active session | In-memory Python list | Session turn initialization |
| **Short-Term Memory** | Store significant recent user interactions and history across conversations|  7 Days (rolling) | Daily JSON files + vector embeddings | 4 triggers (emotion, directive, $\ge 3\times$ repetition, tool failure) via 15-min worker |
| **Long-Term Memory** | Store permanent data such as user preferences, directives, patterns, etc. |Indefinite | Neo4j property graph + partitioned JSON | Directive + keyword gating via 8-hr worker; direct pattern/fix writes from ToolBuilder & Debugger |


---

## Promotion Mechanisms

### Working Memory → Short-Term Promotion (15-Minute Worker)

Working memory holds the session memory in an in-memory buffer. When working memory exceeds its token limit, an inline compaction step automatically summarizes the older conversational half.

In parallel, a background worker runs every 15 minutes during active session. It inspects unprocessed working memory and promotes an entry to short-term memory if any of four triggers fire:

1. **Emotion Strength**: Detected high user urgency, frustration, or excitement.
2. **Explicit Directive**: The turn contains a clear preference or rule.
3. **Repetition**: A turn's embedding matches previous turns with cosine similarity $\ge 0.85$ at least 3 times within a 50-turn window.
4. **Tool Failure**: The entry records a runtime tool fault, preserving context for subsequent debugging.

> **Note**: Signals 1 and 2 are extracted by the Interpreter upon receiving the user input as categorical and boolean variables respectively instead of floating point ranges to avoid noisy value outputs by the LLM.

---

### Short-Term → Long-Term Promotion (8-Hour Worker)

Promoting something to indefinite long-term storage requires much higher confidence. In MAVIS, this is split into two mechanisms:

#### 1. The Background Promotion Worker 
Every 8 hours, a background worker scans short-term turns added since the last scan. Unlike short-term promotion, emotion and repetition alone cannot promote an entry to long-term memory. The following criteria are used for determining whether to promote an entry to long-term memory:
1. Is this a user directive determined by the `is_directive` variable? 
2. Are there explicit keywords such as 'always', 'never', 'remember forever', etc. present in the entry? 

Based on the satisfied criteria, the entry is converted by an LLM into a structured `(subject, predicate, object)` triple and stored in Neo4j.

#### 2. Direct Subsystem Writes 
Certain autonomous components bypass the periodic worker entirely when persisting verified technical knowledge. Once `ToolBuilder` confirms that a synthesized tool passes static analysis and sandbox tests, it immediately calls `write_pattern()`, logging the function signature and conventions to `patterns.json` and creating a tool definition node in Neo4j. Similarly, when `PipelineDebugger` or `AgentBuilder` resolves a tool failure or prompting fault, it immediately calls `write_fix()` / `write_agent_fix()`, logging the verified patch to `fixes.json` and establishing fix relationships in the knowledge graph.

---



## Topic-based Knowledge Graphs

Different components in MAVIS require fundamentally different kinds of memory:
- The **Interpreter** needs user preferences, environment configs, and episodic history to understand user requests.
- The **ToolBuilder** and **PipelineDebugger** need OS specifications, Python runtime details, and prior fix patterns.
- **Background Tasks** need execution state and workflow metadata.

Mixing all of this into a single vector DB results in context cross-contamination and opens up attacks that may target a code-generation component to exfiltrate user data. Interacting with memory is therefore modeled as a **publisher-subscriber model** over partitioned Neo4j graphs and JSON stores. The following table illustrates the components and what they publish and subscribe to.

| Component Namespace | Subscribed Read Topics | Write Topic | User Personal Data Access |
|---|---|---|---|
| **Interpreter** | `["user.*", "env.*", "tooling.*", "agents.*", "debugging.*"]` | `user.profile` | ✅ Permitted |
| **ToolBuilder** | `["env.*", "tooling.*", "debugging.*"]` | `tooling.tools` | ❌ **Hard Blocked** |
| **Debugger** | `["env.*", "tooling.*", "debugging.*"]` | `debugging.fixes` | ❌ **Hard Blocked** |
| **AgentBuilder / AgentDebugger** | `["env.*", "agents.*", "debugging.*"]` | `agents.debugging` | ❌ **Hard Blocked** |
| **PipelineDebugger** | `["env.*", "tooling.*", "agents.*", "debugging.*"]` | `debugging.pipeline_fixes` | ❌ **Hard Blocked** |
| **Background Tasks** | `["env.*", "tooling.*", "tasks.*"]` | `tasks.workflow` | ❌ **Hard Blocked** |

This allows for cross-subsystem memory sharing as any debugging component can access the debugging namespace and learn from previous debugging sessions by other debugging components. For example, `ToolBuilder` can access `debugging.*` namespace and learn from previous debugging sessions by `PipelineDebugger` or `AgentDebugger`. 

This ensures that code-generating components like `ToolBuilder` and `Debugger` are **strictly barred** from querying personal user namespaces (`user.*`). Even if prompt injection targets the tool synthesis prompt, it cannot fetch user data from the agent's memory.


### Resolving Contradictions: Superseding, Not Deleting

User preferences and environment states evolve over time. When a new 1-to-1 functional predicate arrives (e.g., updating `PREFERS_EDITOR = "Neovim"` where `PREFERS_EDITOR = "VS Code"` previously existed), the previous edge is not erased. 

Instead, the graph engine marks the prior relationship edge as inactive with an expiration timestamp. The historical record is preserved for auditing and rollbacks, but subscriber queries filter for active edges only. This prevents vector search's classic failure mode: surfacing two contradictory facts with equal weight.

---

## How Components Use Memory
The final and the most important piece in the memory design is actually ensuring that the memory is properly utilized by the different components. They utilize the memory in the following way for injecting context in the prompt: 

```
Component Query (e.g., user prompt or tool error)
      │
      ├─► Long-Term Memory ─────┬─► Neo4j Graph: Active structured facts (topic-scoped)
      │                         └─► Partitioned JSON: Prior patterns, fixes, & peer records
      │
      ├─► Short-Term Memory ────► Rolling 7-day daily logs (recent interactions & faults)
      │
      └─► Working Memory ───────► Active in-session turns + compaction summaries
      │
      ▼
Unified Injected Prompt Context
```

Therefore, this structured injection is tailored to the needs of each subsystem:

1. **The Interpreter (Intent & Planning)**:
   Before synthesizing a DAG, the Interpreter retrieves active user preferences, environment constraints, and recent conversational turns. If the user previously instructed *"always format git diffs in unified syntax"*, the active fact is injected into the planner prompt, ensuring the generated pipeline obeys the rule without the user having to repeat themselves.

2. **The PipelineDebugger (Fault Recovery)**:
   When a DAG node throws an exception, the debugger queries memory with the failed function name and error message:
   ```python
   memory_priors = memory_store.retrieve_context(
       f"{func_name} {error_message}",
       extra_namespaces=["debugger", "toolbuilder"]
   )
   ```
   By injecting verified fix patterns from past debugging sessions, the debugger avoids repeating known dead-ends and patches runtime errors in a single shot.

3. **The ToolBuilder (Tool Synthesis)**:
   When writing a new tool, it pulls past synthesis patterns and environment specifications, ensuring newly generated code conforms to runtime contracts (like return status codes and error wrappers) without starting from scratch.

---

*In Part 4, I will discuss how MAVIS implements semantic caching to cut latency and cost on redundant queries.*
