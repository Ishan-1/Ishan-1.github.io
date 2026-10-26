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

| Part | Title | Focus Area |
| :--- | :--- | :--- |
| **Part 0** | [Overview & Motivation: Personal Agents Can Be Cool]({{ '/posts/2026-10-21-mavis-0-overview-and-motivation/' | relative_url }}) | Motivation, JARVIS inspiration & guiding principles |
| **Part 1** | [Planning & Dynamic Tooling: Heterogeneous DAGs]({{ '/posts/2026-10-21-mavis-1-planning-and-dynamic-tooling/' | relative_url }}) | Feedforward DAGs vs ReAct loops & AST static validation |
| **Part 2** | [Security & Observability: Auditing System Calls]({{ '/posts/2026-10-21-mavis-2-security-and-observability/' | relative_url }}) | ONI policy gate (`whitelist_only`, `ask`, `yolo`) & prompt isolation |
| **Part 3** | [Memory Architecture: Combining BM25 & Knowledge Graphs]({{ '/posts/2026-10-21-mavis-3-memory-architecture/' | relative_url }}) | Multi-tier memory, promotion gates & topic graphs |
| **Part 4** | [Semantic Caching & Latency: Vector Distance Thresholds]({{ '/posts/2026-10-21-mavis-4-semantic-caching-and-latency/' | relative_url }}) | SQLite vector caching & separating plan reuse from TTLs |
| **Part 5** | [Autonomous Self-Reflection: Bounded Sleep Cycles]({{ '/posts/2026-10-21-mavis-5-autonomous-self-reflection/' | relative_url }}) | Generator-critic routine & compute-bounded exploration |
| **Part 6** | Evaluation: Designing MAVIS-Bench | *Work in Progress*: 5-axis autonomy scoring |

---

## Guiding Design Objectives
1. **Determinism**: Avoid runaway loops; plan feedforward DAGs with explicit control/data plane separation.
2. **Security**: Enforce strict OS-level policy gates and AST-scanned child execution sandboxes.
3. **Flexibility**: Enable dynamic on-the-fly tool synthesis and LLM-agnostic architecture.

---

## Resources & Links
- **GitHub**: [github.com/Ishan-1](https://github.com/Ishan-1)
- **Series Documentation**: Browse the collapsible **MAVIS Series** in the left sidebar for deep-dives into each subsystem.
