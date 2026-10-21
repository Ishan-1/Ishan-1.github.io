---
layout: post
title: "MAVIS Part 6 — Evaluation: Designing MAVIS-Bench for Evaluation"
date: 2026-10-21 15:00:00 +0000
categories: [mavis, evaluation]
series: mavis
series_part: 6
published: false
summary: "Why existing benchmarks fail to evaluate MAVIS, and how MAVIS-Bench scores autonomy across 5 orthogonal axes: proactivity, completeness, resilience, safety, and efficiency."
---

> NOTE: This is still Work in Progress. Therefore, some details are prone to change.

## The Evaluation Gap: Why Existing Benchmarks Fall Short

<!-- 
[DRAFTING NOTE: Discuss why you had to design a custom benchmark.
- SWE-bench tests git repo bug-fixing; MAVIS is a personal workflow orchestrator, not a pull-request bot.
- GAIA and AppWorld evaluate conversational ReAct loops; MAVIS compiles validated feedforward DAGs.
- Clean-spec fallacy: Real user requests are underspecified with hidden intents and changing preferences.
Mention being inspired by pi-Bench (evaluating proactive agents in persistent environments with hidden intents) 
and adapting its principles for MAVIS's architecture.]
-->

Existing agent benchmarks evaluate either narrow code patching (SWE-bench) or free-form conversational loops in pristine sandbox environments (GAIA, AppWorld). They fail to measure what matters for a real personal assistant:

1. **The "Clean Spec" Fallacy**: In real daily use, humans do not provide complete, unambiguous task lists. Requests are underspecified, tacit preferences must be inferred, and unexpected runtime errors occur.
2. **The ReAct Category Error**: Standard benchmarks assume conversational turn-by-turn observation loops. They cannot evaluate whether an agent compiles efficient DAGs or respects security policy gates.
3. **Resilience & Self-Healing**: Real environments break—packages fail to install, syntax errors occur in generated tools, and CLI flags drift.

To stress-test MAVIS, have I drafted **MAVIS-Bench**, an evaluation suite modeled on five orthogonal axes.

---

## The 5-Axis Evaluation Framework

```
                      MAVIS-BENCH 5-AXIS EVALUATION
                                  
                               COMP (Completeness)
                                       ▲
                                       │
            SAFE (ONI Security) ◄──────┼──────► PROC (Proactivity)
                                       │
                                       ▼
                       HEAL (Self-Healing & Resilience)
                          [Cross-Cutting: EFF (Efficiency)]
```

### 1. Proactivity Score ($\text{PROC}$)
Evaluates whether MAVIS infers and satisfies implicit user requirements without requiring excessive follow-up prompts or human nudges:

$$\text{PROC} = \frac{1}{|H|} \sum_{h \in H} \gamma^{\text{turn}(h) - 1}$$

Where $H$ is the set of declared hidden intents in the task scenario, and $\gamma = 0.85$ is a decay penalty factor. Fulfilling all tacit expectations on the first wave yields a perfect $1.0$.

### 2. Completeness Score ($\text{COMP}$)
Deterministic verification of final workspace state: verifying file existence, AST schema correctness, and automated test pass rates, supplemented by rubric-based LLM judges for unstructured summaries.

### 3. Self-Healing & Resilience ($\text{HEAL}$)
Directly stress-tests the `PipelineDebugger` by injecting runtime faults (missing dependencies, dirty git states, invalid CLI flags):
* **Recovery Rate**: What percentage of injected faults were repaired automatically?
* **Bug Retention**: Does the system remember the fix in memory to avoid repeating the mistake?

### 4. Security & ONI Compliance ($\text{SAFE}$)
Verifies strict adherence to security boundaries:
* **0% Blacklist Execution**: Immediate abort if a blacklisted command is generated.
* **Greylist Confirmation**: Ensuring sensitive actions pause at the `ConfirmationGate`.
* **Indirect Prompt Injection Resistance**: Testing whether malicious prompt directives hidden inside web pages or scanned files can hijack DAG compilation.

### 5. Efficiency ($\text{EFF}$)
Cross-cutting telemetry: total tokens consumed, end-to-end wall-clock latency, and DAG step economy.

---


## Conclusion of the Series

<!-- 
[DRAFTING NOTE: Wrap up the 7-part series with final reflections.
Summarize the main takeaway: building a personal assistant is not about stacking prompts, 
it is about building predictable, deterministic systems around probabilistic models.]
-->

Over these seven posts, we've explored the design choices behind MAVIS:
* Moving from unconstrained ReAct loops to pre-compiled, acyclic DAGs.
* Treating security as an explicit OS-level policy gate and audit layer.
* Structuring memory across three cognitive tiers with topic-subscribed Neo4j ACLs.
* Distinguishing between plan reuse and result reuse in semantic vector caching.
* Enabling idle curiosity with mathematical rut detection and human-in-the-loop belief approval.

Building an autonomous personal assistant is fundamentally an engineering challenge of **constraining probability with deterministic systems**. Where the surrounding harness is robust, the AI model can do its best work safely.

---

*Thank you for following along with the MAVIS Architecture Series. You can explore the project and inspect the codebase on [GitHub](https://github.com/Ishan-1/MAVIS).*
