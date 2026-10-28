---
layout: default
title: "Home"
nav_order: 1
description: "Engineering notebook and architecture deep-dives by Ishan Gupta."
permalink: /
---

# Home
{: .fs-9 }

## About
Hi, I am Ishan Gupta. This site is a blog of my software projects and experiments.

---

## MAVIS Architecture Series
An engineering deep-dive series on designing and building **MAVIS** which is a self-modifying personal assistant that can make its own tools and subagents. 

These posts discuss the technical design and choices of various components by first presenting the design problem and then discussing the architectural design to solve the problem.

| Part | Title | Description |
| :--- | :--- | :--- |
| **Part 0** | [Overview & Motivation]({{ '/posts/2026-10-21-mavis-0-overview-and-motivation/' | relative_url }}) | Why build another personal assistant? To experiment on how personal assistants can both be flexible and proactive. |
| **Part 1** | [Planning & Dynamic Tooling]({{ '/posts/2026-10-21-mavis-1-planning-and-dynamic-tooling/' | relative_url }}) | Why MAVIS rejects free-form conversational ReAct loops in favor of acyclic Directed Acyclic Graphs, and how ToolBuilder and AgentBuilder synthesize self-tested tools and agents. |
| **Part 2** | [Security & Observability]({{ '/posts/2026-10-21-mavis-2-security-and-observability/' | relative_url }}) | How do you give an AI assistant system access without handing over full control? How do you observe the different parts of the system? |
| **Part 3** | [Memory Architecture]({{ '/posts/2026-10-21-mavis-3-memory-architecture/' | relative_url }}) | How MAVIS manages context across sessions without catastrophic forgetting: three cognitive tiers, verified promotion criteria, and topic-subscribed Neo4j knowledge graphs. |
| **Part 4** | [Semantic Caching]({{ '/posts/2026-10-21-mavis-4-semantic-caching-and-latency/' | relative_url }}) | How MAVIS cuts latency and token costs: SQLite vector caching, tiered similarity thresholds, separating plan reuse from result TTLs, and deterministic regex pre-filters. |
| **Part 5** | [Autonomous Reflection]({{ '/posts/2026-10-21-mavis-5-autonomous-self-reflection/' | relative_url }}) | Can an AI agent explore on its own and learn? Designing an adversarial generator-critic loop under the principle: exploration is good, but it should be worthwhile. |
| **Part 6** | Evaluation: MAVIS-Bench | *Work in Progress*: Why existing benchmarks fail to evaluate MAVIS, and how MAVIS-Bench scores autonomy across 5 orthogonal axes: proactivity, completeness, resilience, safety, and efficiency. |

---

## Guiding Design Objectives
1. **Determinism**: Avoid runaway loops; plan feedforward DAGs with explicit control/data plane separation.
2. **Security**: Enforce strict OS-level policy gates and AST-scanned child execution sandboxes.
3. **Flexibility**: Enable dynamic on-the-fly tool synthesis and LLM-agnostic architecture.

---

## Resources & Links
- **GitHub**: [github.com/Ishan-1](https://github.com/Ishan-1)
- **Series Documentation**: Browse the collapsible **MAVIS Series** in the left sidebar for deep-dives into each subsystem.
