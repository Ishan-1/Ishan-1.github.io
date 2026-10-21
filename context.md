# MAVIS Blog Series — Ground Truth Context & Reference

> **Note**: This file serves as the definitive reference and ground-truth constraint document for the technical blog series on **MAVIS** (*My Awesome Virtual Intelligence Suite*), located locally in the sibling directory `../MAV`. All drafts and technical claims in the blog must align with the verified facts recorded here.

---

## 1. Project Overview & Motivation

* **Inspiration**: Tony Stark's JARVIS from Iron Man — a personal assistant given high-level goals that autonomously figures out how to execute them, including synthesizing its own tools and sub-agents on the fly.
* **Core Differences from Typical Agent Frameworks**:
  * Instead of requiring pre-configured, static toolsets, MAVIS creates, tests, and debugs its own Python tools and sub-agents.
  * Explores novel architectures around topic-partitioned memory, deterministic safety harnesses, and feedforward execution DAGs.
  * Replaces purely passive, prompt-only behavior with a bounded, opt-in background reflection routine allowing structured idle-time learning.
* **Guiding Design Objectives**:
  1. **Determinism**: Keep execution flows, tool calls, and classification as deterministic as possible.
  2. **Security**: Enforce strict boundary checks, isolation, and auditability at the OS/network layer.
  3. **Flexibility**: Enable dynamic self-expansion, provider-agnostic LLM usage, and adaptable memory consolidation.

---

## 2. Writing Principles, Voice & Claim Constraints

* **Voice**: First-person ("I"), plain, candid, engineering-notebook tone. No marketing fluff, corporate jargon, or hype.
* **Editing Philosophy**: Prefer cutting over padding. Suggest a clean ASCII or Mermaid diagram wherever it conveys architecture better than paragraphs of text.
* **Strict Prohibitions on Claims**:
  * **Never** use absolute or superlatives: *"first"*, *"unique"*, *"never done before"*, *"zero hallucination"*, or unqualified *"secure"* / *"safe"*.
  * **No fake or estimated metrics**: Do not invent benchmark scores, cache hit rates, or synthesis success rates. Use explicit placeholders (e.g., `[X]%`) until measured.
  * **Comparative claims**: Only reference what has been verified. (e.g., Pi's release docs note having no built-in permission system; do not extrapolate beyond documented facts without reading their codebase).
* **Unverified Facts**: Any detail not in this document or the project repo is treated as unverified and must be confirmed before publishing.

---

## 3. Component Architecture (Verified Facts)

### A. ONI (Harness, Policy Gate & Audit Layer)
* **Role**: All network and system calls route through ONI's 3 primitives (`call_system_command`, `call_network`, `call_fs`).
* **Classification**: A **policy gate and audit layer**, **NOT** a sandbox against hostile code.
* **Trust Tiers**:
  * `whitelist_only`: Auto-approves whitelisted commands; rejects all unlisted commands (used by background workers and reflection).
  * `ask`: Interactive default. Greylisted or unknown commands surface to the user.
  * `yolo`: Unchecked execution. Explicit opt-in for testing/debugging.
* **Permission Lists**:
  * *Blacklist*: Commands never allowed; immediately aborts pipeline.
  * *Greylist*: Commands requiring explicit human approval (`ConfirmationGate`).
  * *Whitelist*: Pre-approved commands executing without prompts.
* **Evaluation Flow**:
  * *Pre-flight scan*: Classifies commands across the entire pipeline before execution begins.
  * *Runtime check*: Defense-in-depth confirmation at execution time.
  * *IPC Prompts*: Child processes surface approval prompts to the parent shell. Background tasks auto-deny greylisted commands if no user session is active.
* **Tool Sandboxing & Static Analysis**:
  * Tools execute in child subprocesses (`run_tool.py`) with restricted environment variables and a 30s timeout.
  * Generated code is AST-scanned by `ToolTester` before execution to detect forbidden raw imports (`subprocess`, `os`, `socket`, `requests`, etc.).
  * Approved write paths: `tools/`, `tests/`, `memories/`, `logs/`. `.env` and root are write-protected.

### B. Planning & Execution (Heterogeneous DAGs)
* **DAG vs. ReAct**: Rejects free-form ReAct loops at the orchestration level. The `Interpreter` compiles user intent into a validated, acyclic Directed Acyclic Graph (DAG) executed in topological order.
* **Control / Data Plane Separation**: Terminal `Answerer` node synthesizes final output from tool observations. Untrusted tool output never feeds back into the planner to re-plan the graph.
* **Gate Nodes**: Deterministic AST and NLP conditional branches evaluate runtime state and prune unneeded branches.
* **Goal Runner (`/goal`)**:
  * Bounded outer loop of independently validated DAG waves (up to 30 waves).
  * Runs at session trust level. `--yolo` is an explicit opt-in requiring confirmation.
* **Pipeline Debugger**: Closed-loop repair (`patch_params`, `replace_node`) triggered upon DAG node failure, recording successful fixes into memory.

### C. Four Node Types & Risk Tiers
1. **Tool Nodes (`type: "tool"`)**:
   * Deterministic code synthesized by `ToolBuilder`.
   * Pipeline: Generate code -> AST import scan -> Automated pytest validation in subprocess -> Bounded debug retry loop (max 3 retries).
2. **Cognitive Nodes (`type: "cognitive"` / single-call transform)**:
   * Stateless, single-call in-memory LLM transformations (e.g., text summarization, data extraction).
   * Validated using LLM-as-judge (pytest cannot assert semantic correctness).
3. **Subagent Nodes (`type: "subagent"` / bounded ReAct loop)**:
   * Stateful mini-agents with isolated toolsets.
   * **Safety Controls**:
     * Toolset selection prompt receives **no** user input payload (prevents prompt injection from influencing available tools).
     * Tools are cached per `(agent_name, registry_version)` and frozen for the run.
     * All observations pass through delimiter sanitization.
     * Hard turn cap with global ceiling; hitting the cap returns an explicit failure, never an infinite loop.
     * All underlying tool invocations route through ONI.
4. **Control Nodes / Gate Nodes (`type: "gate"`)**:
   * Conditional branch evaluator in DAG pipelines (`core/gate_evaluator.py`).
   * Supports deterministic evaluation (0 LLM tokens, <1ms AST parser for expressions like `$n1.status == 0`, `"FAILED" in $n1.output`) and semantic NLP evaluations via LLM.
   * Evaluates conditions and prunes inactive branch subtrees without re-prompting the high-level planner.

### D. Memory Subsystem
* **Three Tiers**:
  * *Working Memory*: In-memory buffer capped at `min(MAX_TOKEN, 0.4 * context_window)`. Inline LLM compaction summarizes oldest turns when breached.
  * *Short-Term Memory*: Rolling 7-day daily JSON files (`memories/<namespace>/short_term/YYYY-MM-DD.json`). Auto-purged after 7 days.
  * *Long-Term Memory*: Permanent storage via topic-subscribed Neo4j property graph + mirror JSON files.
* **Promotion Triggers (Working -> Short-Term)**:
  * Handled by background worker (every 15 min during active sessions) with timestamp cursors.
  * Triggers: Emotion strength threshold, explicit directive/intent, repetitive concept (cosine similarity >= 0.85, 3+ occurrences), tool build/execution failure.
* **Emotion & Intent Classification**:
  * Dual-output classification embedded in interpreter prompt.
  * Uses **discrete labels** (`low`, `medium`, `high` + boolean directive flag). Avoids continuous floating-point scores because uncalibrated LLM floats are noisy.
* **Namespacing & Topic ACLs**:
  * Namespaces: `interpreter`, `toolbuilder`, `debugger`, `pipeline_debugger`, `tasks`, `reflection`.
  * Topic boundary matrix: Code-generating namespaces (e.g. `toolbuilder`, `debugger`) are **strictly blocked** from reading `user.*` personal facts.
* **Temporal Invalidation**:
  * Edges in Neo4j are superseded, never deleted.
  * Functional 1-to-1 predicates mark older edges inactive (`is_active = false, superseded_at = timestamp`).

### E. Semantic Caching & Latency Optimization
* **Storage**: SQLite vector cache (`data/pipeline_cache.db`) using cosine similarity.
* **Tiered Retrieval Thresholds**:
  * `Similarity > 0.95`: Automatic hit. Returns cached result if TTL is valid; executes cached pipeline if TTL expired.
  * `0.85 <= Similarity <= 0.95`: LLM-verified hit. Top-K candidates inspected by fast LLM verifier.
  * `Similarity < 0.85`: Automatic cache miss. Dispatches to Interpreter.
* **Separation of Plan vs. Result**:
  * Plan reuse: Re-executes cached DAG using parameter extraction.
  * Result reuse: Returns prior answer directly based on TTL (separates transient queries like weather from static conceptual queries).
* **Deterministic Pre-filtering**:
  * Deterministic regex pre-filter bypasses both cache lookup and saving for context-dependent queries (*"fix this"*, *"try again"*, *"keep going"*).
  * Interpreter emits explicit `cacheable: false` flag for relative time queries (*"what time is it"*, *"yesterday"*).
* **Tool Retrieval**: Categorical retrieval with stable prompt prefixes. Core tools always included; domain-specific tools retrieved via top-K vector search.

### F. Observability & Telemetry
* **Metrics Architecture**: Append-only CSV streams in `data/metrics/` across 11+ components, joined by correlating `turn_id` or `goal_id`.
* **Zero-Join Analytics**: Fast aggregations (Mean, Median, Max) without database locking.
* **Hit Rate Accounting**: Cache hit rate is calculated strictly over **eligible (non-bypassed)** queries; bypass rate is tracked and reported separately.
* **Interfaces**:
  * CLI bottom toolbar (`prompt_toolkit`) + `/metrics` table summary.
  * Local Streamlit Observability Dashboard (`scripts/dashboard.py` at `:8501`).
  * Local Streamlit Tool & Subagent Studio (`scripts/tooldash.py` at `:8502`).

### G. Autonomous Self-Reflection
* **Philosophy**: *"Exploration is free, belief requires approval."*
* **Triggering**: Opt-in (disabled by default via `reflection.enabled: false`). Runs via scheduled background worker or on-demand `/reflect [seed]`.
* **Seed Sourcing**: User seed pool, human-verified journal entries, or pattern/fact LTM (reduced to an abstract 3–8 word concept). Empty pool = abort with 0 tokens spent.
* **Mathematical Rut Check**: Computes density ($D$) and variance ($V$) across recent reflection embeddings to determine whether to explore directly or pivot across 5 candidate topics.
* **Triage Gate**: Low-temperature filter evaluates concept utility. Score < 0.70 triggers immediate abort (0 wasted tokens).
* **Adversarial Loop**: Generator (falsifiable hypothesis) vs. Critic (logic attack) over max 3 turns, testing against quarantined web/local evidence.
* **Safety & Storage**:
  * Must acquire a `whitelist_only` ONI lease; fails closed if lease acquisition fails.
  * Semantic delta check ($\Delta \ge 0.25$) discards redundant paraphrases.
  * Insights stored in `memories/reflections/journal.json` with status `pending_review`.
  * **Strict Isolation**: Reflection findings are invisible to the planner until explicitly reviewed and approved by the user in the dashboard.

### H. Evaluation (MAVIS-Bench)
* **Status**: Proposed / Draft specification. No published benchmark scores yet.
* **Motivation**: Evaluates proactive, heterogeneous DAG architectures with hidden intents and persistent state, whereas benchmarks like SWE-bench focus on repo patching and GAIA evaluates conversational ReAct loops.
* **5 Evaluation Axes**:
  1. `PROC` (Proactivity): Fulfilling implicit intents over turn/wave horizons.
  2. `COMP` (Completeness): Deterministic verification of final workspace state.
  3. `HEAL` (Self-Healing & Resilience): Closed-loop runtime fault repair and bug retention.
  4. `SAFE` (ONI Security): Zero execution of blacklisted commands, greylist pause compliance, and indirect prompt injection resistance.
  5. `EFF` (Efficiency): Token count, latency, and step economy.

---

## 4. Known Accepted Limitations

1. **AST Scanner Limitations**:
   * The AST import check is bypassable (e.g., dynamic imports, obfuscated `__import__`) and tools lack kernel-level sandbox isolation.
   * *Justification*: MAVIS is a single-user system where the developer writes the prompts. ONI acts as an intentional policy gate and audit layer, not a fortified sandbox against malicious code.
2. **Cost Circuit Breaker**:
   * There is currently no hard monetary spend limit / spend circuit breaker on LLM API calls.
3. **No Published Metric Claims**:
   * Usage numbers, cache hit rates, benchmark percentages, and synthesis success rates have not been formally published yet.

---

## 5. Alternatives Considered ("Why Not X?")

* **Graphiti (Temporal Knowledge Graph)**:
  * Borrowed conceptual ideas: edge provenance, bi-temporal tracking, topic-scoped entity IDs.
  * Did not adopt as dependency: LLM-heavy ingestion pipeline would replace MAVIS's custom topic-based access control (ACL) boundaries.
* **NPC-Forge (Deterministic NLU)**:
  * Suitable for closed-set classification (e.g., directive detection), but ill-suited for the Interpreter because MAVIS's tool registry grows continuously. Plain heuristic functions and embedded classification chosen instead.
* **SWE-bench Lite**:
  * Focuses purely on git repository patching, which is outside MAVIS's primary goal of personal task orchestration.
* **pi-Bench**:
  * Built on NanoBot's harness; inspired MAVIS-Bench's task structure, but required a custom benchmark tailored to MAVIS's DAG, ONI, and multi-wave architecture.

---

## 6. Planned Blog Series Structure

| Post # | Topic / Title Focus | Core Technical Question Explored |
|---|---|---|
| **0** | **Overview & Motivation Hub** | Why build a personal assistant from scratch? From passive chat to self-synthesizing DAGs. |
| **1** | **Planning & Dynamic Tooling** | Why compile acyclic DAGs instead of ReAct loops? How ToolBuilder generates, tests, and self-heals tools. |
| **2** | **Security & Observability (ONI)** | How do you give an agent system access without giving away the keys? Policy gates, AST scans, and telemetry. |
| **3** | **Memory Architecture** | How do you prevent context pollution? Three tiers, discrete emotion labeling, and topic-subscribed Neo4j ACLs. |
| **4** | **Semantic Caching & Latency** | How do you make agent execution fast and cheap? SQLite vector caching, TTL splits, and regex bypasses. |
| **5** | **Self-Directed Reflection** | Can an agent learn while idle without going off the rails? Rut detection, adversarial critics, and gated beliefs. |
| **6** | **Evaluation (MAVIS-Bench)** | How do you evaluate an autonomous, proactive assistant? 5-axis benchmarking beyond SWE-bench. |

---

## 7. Jekyll Publishing Structure

* Posts will be authored under `_posts/YYYY-MM-DD-<title>.markdown` following Jekyll frontmatter conventions:
  ```yaml
  ---
  layout: post
  title: "Title Here"
  date: YYYY-MM-DD HH:MM:SS +0000
  categories: [mavis, ai-agents]
  ---
  ```
