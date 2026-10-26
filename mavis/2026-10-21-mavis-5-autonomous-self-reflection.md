---
layout: default
title: "Part 5: Autonomous Reflection"
parent: "MAVIS Series"
nav_order: 6
permalink: /posts/2026-10-21-mavis-5-autonomous-self-reflection/
date: 2026-10-21 14:00:00 +0000
categories: [mavis, architecture]
summary: "Can an AI agent explore on its own and learn? Designing an adversarial generator-critic loop under the principle: exploration is good, but it should be worthwhile."
---

# Part 5: Autonomous Reflection
{: .fs-8 }

Can an AI agent explore on its own and learn? Designing an adversarial generator-critic loop under the principle: exploration is good, but it should be worthwhile.
{: .fs-5 .fw-300 .text-grey-dk-000 }

---

## Why so Idle?

A few months back, I found out the full version of the proverb about curiosity: *"Curiosity killed the cat, but satisfaction brought it back."*

This describes what I have always felt is missing in many AI assistants: they only learn when you interact with them or explicitly ask them to explore. They never explore on their own downtime to learn things that could help them become better at their tasks. I wanted to give that capability to MAVIS, but with guardrails, because the internet is a wild place and idle wandering can easily turn into an infinite token burn.

I designed the **Autonomous Reflection Engine** around one foundational principle:

> **"Exploration is good but it should be worthwhile."**

---

## System Architecture

```
[ Trigger: Daily Worker / /reflect ]
               │
               ▼
┌──────────────────────────────────────────────┐
│ Phase 1: Topic Selection & Rut Detection     │
│ - Sample seed pool / long-term memory        │
│ - Reduce to 3-8 word technical concept       │
│ - Compute embedding density (D) & variance(V)│
│ - Rut check: Pivot to novel candidate?       │
└──────────────────────┬───────────────────────┘
                       │ Selected Concept
                       ▼
┌──────────────────────────────────────────────┐
│ Phase 2: The Triage Gate (Categorical Filter)│
│ - 4-tier rubric evaluation (1.00/0.75/0.40/0)│
│ - Score < 0.70 ──► Instant Abort (0 tokens)  │
└──────────────────────┬───────────────────────┘
                       │ Score ≥ 0.70
                       ▼
┌──────────────────────────────────────────────┐
│ Phase 3: Adversarial Exploration Loop        │
│ - Generator prompt: Formulate hypothesis     │
│ - Dual-source evidence (quarantined web/docs)│
│ - Critic prompt: Logic attack & falsification│
│ - Max 3 turns (ACCEPT / REFINE / REJECT)     │
└──────────────────────┬───────────────────────┘
                       │ Validated Insight
                       ▼
┌──────────────────────────────────────────────┐
│ Phase 4: Semantic Delta Check & Storage      │
│ - Discard if semantic delta Δ < 0.25         │
│ - Save to journal with status: pending_review│
└──────────────────────────────────────────────┘
```

---

## Phase 1: Topic Selection and Rut Detection

To start exploring, it is important to pick the seed topic. The reflection engine maintains a pool of seed topics from which it can sample. Seeds are sourced by either selecting recent shards of long-term memory or selecting manually adding seeds through the reflections tab in the dashboard. In case of selecting a long-term memory shard, the concept is explicitly extracted in 3–8 words instead of using the entire memory shard as the seed.

But this does not guarantee that the topic selected is new. To prevent this, rut detection is performed against reference vectors—specifically, the top 10 nearest vector embeddings drawn from active working memory and the last five days of reflection journals:
- **Embedding Density ($D$)**: Measures how densely the reference embeddings are clustered together. A high density indicates that the agent is already familiar with the topic.
- **Embedding Variance ($V$)**: Measures the spread of the reference embeddings. A low variance indicates a tight, repetitive cluster.

When $D$ is high and $V$ is low, the seed is considered to be in a rut. In such cases, an LLM is used to generate 5 neighboring topics that are randomly selected with an inverse softmax weighting based on similarity to the current seed. This ensures that the agent explores topics that are related to its current knowledge but not too close to what it already knows. 

---

## Phase 2: Topic Viability Check (The Triage Gate)

Selecting a seed topic in Phase 1 ensures novelty, but it does not guarantee utility. Before exploration, MAVIS checks its viability using a low-temperature LLM evaluator that scores the seed against four discrete anchored utility tiers:

```
- 1.00 (High Utility): Directly yields concrete operational rules, CLI flags, DAG execution heuristics, error recovery procedures, or explicit assistant interaction policies.
- 0.75 (Actionable Pattern): Specific API convention, systems heuristic, or validated human-assistant cognitive pattern with clear practical utility.
- 0.40 (Marginal / General Theory): High-level abstract theory, general philosophy, or generic advice with no concrete operational translation.
- 0.00 (Reject / Trivia): Generic trivia, empty self-help platitudes, surface-level definitions, subjective opinion, or common knowledge.
```

Because the passing threshold is set to $0.70$, this check passes for only seeds evaluated as an *"Actionable Pattern"* ($0.75$) or *"High Utility"* ($1.00$) are viable for exploration. Topics that amount to abstract philosophy or trivia are rejected.

---

## Phase 3: Exploration Loop

Exploration is done in an adversarial loop where two subagents, Generator and Critic, work against each other. Generator tries to generate hypotheses from the seed and gather evidence from web and local codebase. Critic reviews the hypothesis and the evidence and tries to find logic flaws and vulnerabilities in the hypothesis and the evidence. The Generator tries to address the flaws and tries to generate a better hypothesis and gather better evidence. This loop continues until the critic accepts the hypothesis or the iteration limit is reached, at which point if the critic hasn't rejected the hypothesis, it is considered accepted and proceeds to Phase 4.

---

## Phase 4: Semantic Delta Check and Storage

Even if an insight survives the adversarial loop, it must be checked against existing knowledge to prevent duplicate entries. The novelty of the synthesis is measured using the Semantic Delta:

$$\text{Semantic Delta } (\delta) = \max\Big(0, 1 - \max_{s \in \text{ref\_vectors}} \text{similarity}(\text{synthesis}, s)\Big)$$

Where $\text{ref\_vectors}$ represents the nearest reference embeddings from working memory and recent self-reflections. If $\delta < 0.25$, the insight is deemed a paraphrase of existing knowledge and discarded.

If $\delta \ge 0.25$, the entry is persisted to a JSON with a pending status/

---

## Safety Measures in Self-Reflection

Background reflection runs under strict security constraints:

* **Mandatory `whitelist_only` Lease**: The reflection worker must acquire a `whitelist_only` approval from ONI. If it cannot be granted, it fails. This prevents unintended behaviour by the system.
* **Quarantined Evidence**: Web search results and local file observations are wrapped in strict quarantine delimiters, preventing prompt injection attacks from hijacking the critic or generator prompts.
* **Human-in-the-Loop Journal**: All generated insights remain quarantined with `verified: false`. They cannot be retrieved by the `Interpreter` or used for DAG planning until the user explicitly reviews and approves them in the Streamlit dashboard.

---

## Concluding the Architecture Series

This post wraps up the core architecture series for MAVIS. As I mentioned at the start of the series, the project still has a long way to go in terms of both design and implementation to become a truly autonomous personal assistant. The final post on the project will focus on how to evaluate MAVIS and why normal benchmarks fail to do so. The proposed benchmark is a work in progress and will be published soon.


---

<div style="display: flex; justify-content: space-between; align-items: center; margin-top: 2rem; padding-top: 1rem; border-top: 1px solid #eee; font-size: 0.9rem;">
  <div>← Previous<br><a href="{{ "/posts/2026-10-21-mavis-4-semantic-caching-and-latency/" | relative_url }}"><strong>Part 4 — Semantic Caching</strong></a></div>
  <div style="text-align: right;">Next →<br><a href="{{ "/posts/2026-10-21-mavis-6-evaluation-mavis-bench/" | relative_url }}"><strong>Part 6 — Evaluation: MAVIS-Bench</strong></a></div>
</div>
