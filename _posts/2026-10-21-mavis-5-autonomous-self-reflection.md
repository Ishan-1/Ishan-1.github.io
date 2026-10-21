---
layout: post
title: "MAVIS Part 5 — Autonomous Reflection: Curiosity, Rut Detection, and Gated Beliefs"
date: 2026-10-21 14:00:00 +0000
categories: [mavis, architecture]
series: mavis
series_part: 5
summary: "Can an AI agent explore on its own and learn? Designing an adversarial generator-critic loop under the principle: exploration is good, but it should be worthwhile."
---

## Why so Idle?

A few months back, I found out the full version of the proverb about curiosity: *"Curiosity killed the cat, but satisfaction brought it back."*

This describes what I have always felt is missing in many AI assistants: they only learn when you interact with them or explicitly ask them to explore. They never explore on their own downtime to learn things that could help them become better at their tasks. I wanted to give that capability to MAVIS, but with guardrails, because the internet is a wild place and idle wandering can easily turn into an infinite token burn.

I designed the **Autonomous Reflection Engine** around one foundational principle:

> **"Exploration is good but it should be worthwhile."**

Idle exploration must lead to concrete satisfaction—learning a new pattern, resolving a recurring failure, or compressing an active rule—rather than aimless rabbit holes. And as a security precaution against memory pollution, **no reflection finding is ever visible to the planner** until the user reviews and approves it in the dashboard.

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
│ Phase 2: The Triage Gate (Low-Temp Filter)   │
│ - Score utility for rule/compression         │
│ - Score < 0.70 ──► Instant Abort (0 tokens)  │
└──────────────────────┬───────────────────────┘
                       │ Score ≥ 0.70
                       ▼
┌──────────────────────────────────────────────┐
│ Phase 3: Adversarial Exploration Loop        │
│ - Generator: Testable, falsifiable hypothesis│
│ - Dual-source evidence (quarantined web/docs)│
│ - Critic: Logic attack & falsification check │
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

To start exploring, it is important to pick the seed topic. The reflection engine has a pool of seed topics from which it can pick. The seeds are sourced by either selecting recent shards of long-term memory or by selecting manually added seeds that can be added through the dashboard. In case of selecting a long-term memory shard, the concept is explicitly extracted in 3-8 words instead of just using the entire memory shard.

But this does not guarantee that the topics selected are novel. The same concepts might be selected repeatedly over time. To prevent this, rut detection is done by calculating the following metrics:
- Embedding Density ($D$): This measures how densely the embeddings of recent reflection topics are clustered together. A high density indicates that the agent is exploring familiar topics.
- Embedding Variance ($V$): This measures the variance of the embeddings of recent reflection topics. A low variance indicates that the agent is exploring familiar topics.

When $D$ is high and $V$ is low, the seed is considered to be in a rut. In such cases, an LLM is used to generate 5 neighboring topics that are randomly selected with an inverse softmax weighting based on similarity to the current seed. This ensures that the agent explores topics that are related to its current knowledge but not too close to what it already knows. 

---

## Phase 2: Topic Viability Check

Selecting a seed topic in Phase 1 ensures novelty, but novelty alone does not guarantee utility. So, before exploring the seed topic, it is important to check its viability. This is done by using a low-temperature LLM evaluation where the seed is assigned a utility score based on how likely it is to yield useful information. The following rubric is utilized:

```
- 1.00 (High Utility): Directly yields concrete operational rules, CLI flags, DAG execution heuristics, error recovery procedures, or explicit assistant interaction policies (e.g., communication pacing, memory retention thresholds, confirmation vs. execution boundaries).
- 0.75 (Actionable Pattern): Specific API convention, systems heuristic, or validated human-assistant cognitive pattern (e.g., progressive disclosure, task decomposition model, cognitive load reduction) with clear practical utility.
- 0.40 (Marginal / General Theory): High-level abstract theory, general philosophy, or generic advice with no concrete operational translation.
- 0.00 (Reject / Trivia): Generic trivia, empty self-help platitudes, surface-level definitions, subjective opinion, or common knowledge.
```
If the utility score meets or exceeds the threshold (typically $\ge 0.70$), the seed advances to exploration; otherwise, it is immediately aborted with zero further tokens spent.

---

## Phase 3: Exploration Loop

Exploration is done in a loop where two subagents, Generator and Critic, work against each other. Generator tries to generate hypotheses from the seed and gather evidence from web and local codebase. Critic reviews the hypothesis and the evidence and tries to find flaws in the hypothesis and the evidence. The Generator tries to address the flaws and tries to generate a better hypothesis and gather better evidence. This loop continues until the critic accepts the hypothesis or the iteration limit is reached, at which point if the critic hasn't rejected the hypothesis, it is considered accepted and proceeds to Phase 4.

---

## Phase 4: Semantic Delta Check and Storage

The novelty of the final hypothesis is measured using the following formula:

$$\text{Semantic Delta } (\delta) = \max\Big(0, 1 - \max_{s \in \text{memory}} \text{similarity}(\text{hypothesis}, s)\Big)$$

If the delta exceeds the threshold (e.g., $\delta \ge 0.25$), the hypothesis is considered novel and is stored in long-term memory as a new shard. If the delta is below the threshold, the hypothesis is discarded.

---

## Safety Measures in Self-Reflection

Background reflection runs under strict security constraints:

* **Mandatory `whitelist_only` Lease**: The reflection worker must acquire a `whitelist_only` lease from ONI. If the lease cannot be granted, it fails closed. It cannot run shell commands or write outside its dedicated reflection memory directory.
* **Quarantined Evidence**: Web search results and local file observations are tagged with quarantine delimiters, preventing indirect prompt injections from hijacking the critic.
* **Human-in-the-Loop Journal**: All output is appended to `memories/reflections/journal.json` marked as `pending_review` and cannot be read by the `Interpreter` until they are explicitly verified in the Streamlit dashboard.

---

## Concluding the Architecture Series

This post wraps up the core architecture series for MAVIS. As I mentioned at the start of the series, the project still has a long way to go in terms of both design and implementation to become a truly autonomous personal assistant. The final post on the project will focus on how to evaluate MAVIS and why normal benchmarks fail to do so. The proposed benchmark is a work in progress and will be published soon.
