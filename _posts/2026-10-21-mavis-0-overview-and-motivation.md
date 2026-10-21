---
layout: post
title: "MAVIS Part 0 — Overview & Motivation: Personal Agents Can Be Cool"
date: 2026-10-21 09:00:00 +0000
categories: [mavis, architecture]
series: mavis
series_part: 0
summary: "Why build another personal assistant? To experiment on how personal assistants can both be flexible and proactive."
---


## Motivation
MAVIS is inspired from how JARVIS was portrayed in the Iron Man movies, a personal assistant who you just have to give a task and let it just do it. While this project certainly doesn't come close, it's an attempt in that direction. The AI agents I usually see require you to give them the tools beforehand and I wanted to try letting it make its own tools and subagents. Moreover, there were some ideas I wanted to experiment on how to design harness and memory. Another thing I noticed that many assistants are passive in the sense that they do stuff only when you prompt it and I wanted to see how a routine can be designed that allows the agent to learn stuff on its own. 

**Most importantly, I just wanted to build an agent that I would want to personally use.**

## Guiding Design Objectives
I have tried to adhere to the following objectives when designing MAVIS:
1. **Keeping it as deterministic as possible**: While most agents usually work on loops, I wanted to ensure that loops are avoided whenever possible and that planning is done in a more linear manner to reduce both cost and uncertainty. 
2. **Keep it as secure as possible**: I have seen online posts on how AI agents are prone to making mistakes which can be fatal such as deleting your files, mail inbox, etc. randomly without permission or can be prone to prompt injections from external content. Therefore, I felt that it is necessary to ensure that security is not a mere afterthought for something that is supposed to interact with my computer.
3. **Make it as flexible as possible**: MAVIS only has some 3-4 core tools/agents that are required for it to operate properly out-of-the-box and that it will adapt to user's need as time goes on. Moreover, I have kept the project LLM-agnostic so models can be switched whenever needed without losing any context or functionality. 


## High-Level Architecture

MAVIS separates the user interaction shell, the planning engine, dynamic tool creation, memory storage, and the execution harness into distinct components:


```
User Input
    │
    ▼
[ Semantic Cache Check ] ──(Hit similarity >0.95)──► Instant Reply / Plan Re-execution
    │
    ▼ (Miss)
[ Interpreter ] ─────────► Compiles Heterogeneous Feedforward DAG
    │
    ├─► Missing tools? ──► [ ToolBuilder ] (Generate -> AST Scan -> Pytest -> Debug)
    │
    ▼
[ DAG Executor ] ────────► Runs nodes in topological order via ONI policy gate
    │
    ▼
[ Answerer ] ────────────► Synthesizes final response (Data plane isolated from Planner)
```
The basic flow is this: You run the MAVIS CLI, give it a task, the interpreter compiles a plan in the form of a DAG, figures out if it needs to create any tools or subagents, creates them using ToolBuilder or AgentBuilder, tests them, and finally executes them to complete the task once the required permissions have been granted by the user which is handled by ONI (OS & Network Interface). The other components work in background; there are memory workers that handle memory extraction and promotion, a task scheduler takes care of scheduled tasks and more. 

---

## Known Limitations
Before diving into the rest of the series, it is important to be clear about what MAVIS does **not** claim:
* **It is not for everyone**: Since I have designed MAVIS for myself, it will not be comfortable to use for everyone. It expects that the user is comfortable with CLIs, Linux and knows the basics.
* **It is not a specialist agent**: It is a personal agent and can be used for a lot of tasks such as software development, summarizing mails, etc. but it is not explicitly designed for any specific task. 
* **ONI is a policy gate and audit layer, not a fortified sandbox against malicious code**: The AST scanner can be bypassed (e.g. through obfuscated imports), and generated tools run in local subprocesses without kernel-level sandboxing. Because MAVIS is a single-user system where I write the prompts and MAVIS tool builder usually handles tool creation, this is an acceptable trade-off.
* **No benchmark claims yet**: This is still a Work In Progress. There are no existing benchmarks I can directly use so I am working on one.

---

## The Roadmap for This Series

This blog series is structured as a design problem introduced at the start of each post and how MAVIS is designed to solve them:

* **[Part 1: Planning & Dynamic Tooling: Feedforward DAGs, ToolBuilder and AgentBuilder]({{ '/posts/2026-10-21-mavis-1-planning-and-dynamic-tooling/' | relative_url }})** — Why MAVIS rejects free-form conversational ReAct loops in favor of acyclic Directed Acyclic Graphs, and how ToolBuilder and AgentBuilder synthesize self-tested tools and agents.
* **[Part 2: Security & Observability: Policy Gates and Metrics Dashboard]({{ '/posts/2026-10-21-mavis-2-security-and-observability/' | relative_url }})** — How do you give an AI assistant system access without handing over full control? How do you observe the different parts of the system?
* **[Part 3: Memory Architecture: 3 Tiers, Discrete Labels, and Topic-based Knowledge Graphs]({{ '/posts/2026-10-21-mavis-3-memory-architecture/' | relative_url }})** — How MAVIS manages context across sessions without catastrophic forgetting: three cognitive tiers, verified promotion criteria, and topic-subscribed Neo4j knowledge graphs.
* **[Part 4: Semantic Caching: Plan vs. Result Reuse]({{ '/posts/2026-10-21-mavis-4-semantic-caching-and-latency/' | relative_url }})** — How MAVIS cuts latency and token costs: SQLite vector caching, tiered similarity thresholds, separating plan reuse from result TTLs, and deterministic regex pre-filters.
* **[Part 5: Autonomous Reflection: Curiosity, Rut Detection, and Gated Beliefs]({{ '/posts/2026-10-21-mavis-5-autonomous-self-reflection/' | relative_url }})** — Can an AI agent explore on its own and learn? Designing an adversarial generator-critic loop under the principle: exploration is good, but it should be worthwhile.
* **Part 6: Evaluation: Designing MAVIS-Bench for Evaluation** *(Coming Soon)* — Why existing benchmarks fail to evaluate MAVIS, and how MAVIS-Bench scores autonomy across 5 orthogonal axes: proactivity, completeness, resilience, safety, and efficiency.

> NOTE: There are some minor QoL features such as Tool dashboard where you can manage your tools and sub-agents, slash commands in CLI, etc. that are not discussed because they are quite trivial. You will have to read the repo README and docs folder if you are interested.

---

*In Part 1, I will discuss how MAVIS handles task planning and execution along with tool and agent synthesis.*
