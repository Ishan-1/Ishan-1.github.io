---
layout: default
title: "Part 1: Planning & Dynamic Tooling"
parent: "MAVIS Series"
nav_order: 2
permalink: /posts/2026-10-21-mavis-1-planning-and-dynamic-tooling/
date: 2026-10-21 10:00:00 +0000
categories: [mavis, architecture]
summary: "Why MAVIS rejects free-form conversational ReAct loops in favor of acyclic Directed Acyclic Graphs, and how ToolBuilder and AgentBuilder synthesize self-tested tools and agents."
---

# Part 1: Planning & Dynamic Tooling
{: .fs-8 }

Why MAVIS rejects free-form conversational ReAct loops in favor of acyclic Directed Acyclic Graphs, and how ToolBuilder and AgentBuilder synthesize self-tested tools and agents.
{: .fs-5 .fw-300 .text-grey-dk-000 }

---

## The Flaw of Unbounded ReAct Loops
In most agent architectures, the agent runs an open-ended loop: `Thought -> Action -> Observation -> Thought`. While flexible, this pattern suffers from the following issues:
* **Non-determinism**: You cannot inspect what the agent plans to do before it starts doing it.
* **Runaway loops**: Retries and self-corrections can cycle endlessly without hard architectural boundaries.
* **Control-plane pollution**: Untrusted output from tools (like a web page or file) feeds directly back into the planner prompt.

The three of them violate the first 2 design objectives, therefore, I needed a different approach. Enter Directed Acyclic Graphs (DAGs).

---

## Using Validated DAGs for planning

Instead of free-form conversation loops, MAVIS's `Interpreter` treats the user prompt as an input task and creates an acyclic **Directed Acyclic Graph (DAG)** to accomplish that task:

```
User Prompt
     │
     ▼
[ Interpreter ] ─────────► Generates DAG Specification (JSON)
     │
     ├─► Topological Sort & Acyclicity Validation
     ├─► Pre-flight ONI Security Scan
     │
     ▼
[ DAG Engine ] ──────────► Executes Nodes in Topological Order
     │
     ▼
[ Answerer Node ] ───────► Synthesizes Final Presentation
```

### Key Architectural Properties
1. **Topological Sort**: Dependencies are mathematically resolved using topological sort before the first tool runs.
2. **Bounded Execution**: Since the graph is directed and acyclic, it mathematically guarantees termination. 
3. **Data-Plane Isolation (The Answerer Node)**: Raw tool outputs flow downstream to dependent nodes and finally into a dedicated `Answerer` node. Untrusted tool output **never** loops back to the high-level planner within a single DAG, avoiding the control-plane injection risks associated with free-form ReAct loops.

---

## DAG Node Types

MAVIS categorizes every DAG node into one of four execution types:

| Node Type | Execution Model | How It Is Tested |
|---|---|---|
| **Tools** | Deterministic local Python function | Automated `pytest` in isolated subprocess + AST import scanner |
| **Cognitive** | Stateless, single-call LLM transform | LLM-as-judge (pytest cannot assert semantic correctness) |
| **Subagents** | Bounded ReAct loop with scoped tools | Isolated harness with simulated tools & turn ceiling |
| **Control (Gate)** | Conditional branching node (AST or NLP) | Deterministic AST expression parser (0 LLM tokens) or LLM evaluator|

While deterministic tools, single-call cognitive nodes, and control gates are relatively straightforward, subagents require special handling. A subagent runs a mini ReAct loop for tasks that are inherently exploratory, but allowing free-form loops inside an assistant risks the exact non-determinism and injection vulnerabilities MAVIS should avoid. 

To keep subagents bounded and safe, MAVIS enforces four constraints at runtime:

1. **Payload-Blind Tool Selection**: Before a subagent begins, it must declare its tool requirements. The prompt that selects these tools only receives the subagent's role and task specification, not the raw user input payload. This ensures that prompt injection cannot in the input cannot trick the agent into requesting dangerous tools.
2. **Frozen Scoped Toolsets**: The declared toolset is cached and frozen for execution. Subagents cannot call or discover any tools outside this pre-approved scope, and sensitive or greylisted operations trigger interactive confirmation prompts at call time.
3. **Quarantined Observations**: Any data returned from tools (such as web pages or file contents) is sanitized and wrapped in strict quarantine delimiters, mitigating tool outputs from acting as system prompts.
4. **Hard Turn Ceilings**: Every subagent has a strict limit on how many ReAct iterations it can execute. If it fails to complete the task within that ceiling, it returns an explicit failure status and terminates.

---

## Dynamic Synthesis: ToolBuilder and AgentBuilder
When the Interpreter plans a DAG, it often encounters capabilities that do not yet exist in its registry. Rather than failing or requiring manual tool development, MAVIS synthesizes the missing capability on the fly.

Because deterministic code execution and cognitive reasoning have completely different building and testing requirements, MAVIS splits synthesis into two specialized builders: **ToolBuilder** (for deterministic Python functions) and **AgentBuilder** (for subagents).

### 1. ToolBuilder: Synthesizing Deterministic Code

When the required capability is deterministic (such as file parsing, API queries, or data transformation), `ToolBuilder` writes, tests, and self-debugs Python modules:

```
Interpreter identifies missing capability
                  │
                  ▼
         [ ToolBuilder.build_tool() ]
                  │
                  ▼
         [ 1. Python Code Generation ]
                  │
                  ▼
         [ 2. Static AST Import Scan ] ──(Forbidden import)──► [ ToolDebugger ]
                  │ (Passed)                                        ▲
                  ▼                                                 │
         [ 3. Automated Pytest in Subprocess ] ──(Assertion fail)───┘
                  │ (Passed)
                  ▼
         [ 4. Register in Catalog & Neo4j ]
```

1. **Code Generation**: `ToolBuilder` retrieves past tooling patterns from long-term memory (`memories/toolbuilder/`) and generates a self-contained Python module.
2. **Static AST Import Scan**: Before any generated code is executed, `ToolTester` parses its Abstract Syntax Tree (AST) to check for forbidden imports and shell-execution calls (`subprocess`, `socket`, `os.system`, `os.popen`, etc.). If an unapproved import or execution call is detected, the build fails immediately.
3. **Automated Pytest in Isolated Subprocess**: `ToolBuilder` writes companion test cases and executes `pytest` inside a child subprocess (`run_tool.py`) with a 30-second hard timeout.
4. **ToolDebugger Retry Loop**: If tests fail or syntax errors occur, the failure traceback is fed into `ToolDebugger` for bounded retries (up to 3 attempts). Successful fixes are recorded in memory so future builds avoid the same bug.
5. **Registration**: Once all tests pass, the tool is registered in `commands_list.json` and indexed in Neo4j.

---

### 2. AgentBuilder: Synthesizing Subagents
When the required capability requires subjective reasoning, semantic data manipulation, or multi-turn tool orchestration, `AgentBuilder` synthesizes a subagent conforming to `BaseAgent`.

Testing an agent cannot be done with `pytest`—LLM outputs are probabilistic, string assertions are brittle, and subagents execute multi-turn loops. Instead, `AgentBuilder` pairs with **`AgentTester`**, which evaluates candidate subagents using an **LLM-as-a-Judge** harness:

1. **Simulated Tool Execution**: Subagents need tools to execute their ReAct loops. To prevent testing from executing live, destructive, or unapproved OS commands, `AgentTester` injects a simulated tool execution harness during the test run.
2. **Synthetic Multi-Turn Test Cases**: `AgentTester` prompts the LLM to generate 2–3 realistic test scenarios matching the subagent's declared input and output schemas, covering both standard tasks and edge cases.
3. **Execution Verification**: The candidate subagent runs its ReAct loop against each test case. It must successfully coordinate its simulated tools and terminate with a clean exit code (`0`) within its hard turn limit.
4. **Discrete Binary Evaluation (LLM-as-a-Judge)**: A dedicated Judge prompt inspects the subagent's final output. To avoid noisy floating-point scores, the Judge returns a strict binary verdict: **`passed`** or **`failed`**, along with an explicit diagnostic reason.
   - *Rubric*: Schema compliance, factual consistency, adherence to negative constraints (no conversational fluff), and instruction containment.
5. **Prompt Refinement via AgentDebugger**: If the subagent fails its test cases, `AgentDebugger` feeds the failure diagnostic and test input back into a targeted prompt-refinement loop (up to 3 retries). It adjusts system instructions and negative constraints rather than patching Python code, and logs successful fixes into memory so future subagent builds learn from past failures.

---

## Long-Horizon Goals: The Goal Runner (`/goal`)

While simpler tasks—such as scraping data, summarizing text, or making specified edits—can be executed within a single feedforward DAG, complex objectives require an iterative loop: discovering state, taking action, evaluating results, and planning next steps. 

For example, consider a codebase refactoring task:
1. **Wave 1 (Discovery)**: Inspect the project structure, locate dependencies, and identify code smells.
2. **Wave 2 (Execution)**: Refactor the identified functions and update call sites.
3. **Wave 3 (Verification)**: Run tests and linters to verify nothing broke.

In a conventional agent, this would be tossed into an open-ended ReAct loop. In MAVIS, the `/goal` runner executes this as a sequence of **independent DAG waves**:

```
High-Level User Goal: "/goal Refactor auth module"
                 │
                 ▼
     ┌───────────────────────┐
     │ Wave 1: Discovery DAG │ ──► (Inspect files, analyze AST)
     └───────────┬───────────┘
                 │ Wave 1 Results & State
                 ▼
     ┌───────────────────────┐
     │ Wave 2: Execution DAG │ ──► (Patch code, update imports)
     └───────────┬───────────┘
                 │ Wave 2 Results & State
                 ▼
     ┌───────────────────────┐
     │ Wave 3: Verify DAG    │ ──► (Run test suite, verify clean exit)
     └───────────┬───────────┘
                 │ Goal Evaluator: "Tests pass, goal complete."
                 ▼
            Final Output
```

### Safety and Termination Guarantees
1. **Mathematical Acyclicity per Wave**: Each wave is independently compiled and strictly acyclic. It cannot deadlock or get trapped in a cyclic edge.
2. **Bounded Iteration Ceiling**: The outer loop has a hard iteration cap. If the goal is not met when the ceiling is reached, the runner halts and reports the partial progress rather than burning tokens indefinitely.
3. **Trust & ONI Leases**: Goal waves run at the session's active trust level. If an unapproved or greylisted tool is encountered, it pauses for user confirmation. Running unattended requires the `--yolo` flag, which prompts for an explicit confirmation before starting. 
4. **Cross-Wave State Sharing**: While a single DAG completely isolates untrusted tool observations from the planner, multi-wave goals carry forward high-level wave outcome summaries and blackboard notes so the next wave planner knows what was accomplished.

---

*In Part 2, I will discuss how MAVIS actually executes these tools safely: the ONI harness, subprocess isolation, and append-only audit logs.*


---

<div style="display: flex; justify-content: space-between; align-items: center; margin-top: 2rem; padding-top: 1rem; border-top: 1px solid #eee; font-size: 0.9rem;">
  <div>← Previous<br><a href="{{ "/posts/2026-10-21-mavis-0-overview-and-motivation/" | relative_url }}"><strong>Part 0 — Overview & Motivation</strong></a></div>
  <div style="text-align: right;">Next →<br><a href="{{ "/posts/2026-10-21-mavis-2-security-and-observability/" | relative_url }}"><strong>Part 2 — Security & Observability</strong></a></div>
</div>
