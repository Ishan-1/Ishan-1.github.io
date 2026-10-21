---
layout: post
title: "MAVIS Part 2 — Security & Observability: Policy Gates and Metrics Dashboard"
date: 2026-10-21 11:00:00 +0000
categories: [mavis, architecture]
series: mavis
series_part: 2
summary: "How do you give an AI assistant system access without handing over full control? How do you observe the different parts of the system?"
---

## Freedom Without Chaos
To be useful, an AI assistant must be able to interact with the system and network. But giving it too much freedom is dangerous as it possesses the responsibility of a child with access to a sharp object. Therefore, a strong policy layer is required to ensure that the assistant remains within the bounds of what it is supposed to do. To solve this, I designed **ONI** (Operating System & Network Interface) for MAVIS that acts as an explicit policy gate and audit layer that intercepts all system and network calls.

---

## ONI: Three Primitives and Two Evaluation Phases

No tool, background worker, or subagent in MAVIS is permitted to import Python's standard `os`, `subprocess`, `socket`, or `requests` modules directly. Instead, all I/O is channeled through three primitive functions:

* `call_system_command(command, params)` — executes sanctioned OS-level commands.
* `call_network(url, data, method, ...)` — executes all outbound HTTP/network requests.
* `call_fs(operation, path, data, ...)` — controls file system reads, writes, deletes, and package installations.

### Two-Phase Trust Evaluation

To avoid stopping execution in the middle of a multi-node DAG, ONI evaluates permissions in two phases:

```
[ Pre-Flight Scan ] ──────────────► Scans all planned DAG nodes upfront
        │
        ├─► Any Blacklisted? ─────► Abort entire pipeline immediately
        ├─► Any Greylisted? ──────► Prompt user once with batch approval queue
        │
        ▼ (All approved)
[ Pipeline Execution ] ───────────► Nodes execute in topological order
        │
        ▼
[ Runtime Check ] ────────────────► Second-line defense-in-depth at call time
```

### Trust Tiers
* **`whitelist_only`**: Auto-approves whitelisted tools and silently denies all others. Used by background workers and autonomous reflection to ensure they never run unapproved operations.
* **`ask`** (Default): Greylisted operations (such as installing third-party packages or modifying critical directories) surface a human-in-the-loop confirmation prompt.
* **`yolo`**: Executes commands without interactive confirmation prompts. Intended strictly as an explicit opt-in for tool building and test cycles.

---

## Tool Sandboxing & AST Static Analysis
Generated tools run in a separate **child subprocess** using a lightweight wrapper that helps achieve the following:

1. **Subprocess Isolation**: If a generated tool crashes, segfaults, or times out (30s hard limit), the parent MAVIS process remains completely unharmed.
2. **Restricted Environment**: Sensitive API keys and environment variables not declared by the tool are stripped from the child process.
3. **AST Import Scanner**: Before any synthesized tool code is executed, `ToolTester` parses its Python Abstract Syntax Tree (AST) to check for forbidden imports (`subprocess`, `os`, raw network calls). If found, the build fails before any code executes.

---
## Accepted Limitation: Policy & Audit, Not Complete Sandboxing

> **Design Boundary**: ONI is designed as an intentional **policy gate and audit layer**, *not* a bulletproof sandbox against hostile, malicious code. An attacker with arbitrary code injection could bypass the AST scanner via dynamic runtime imports. Because MAVIS is a single-user personal tool designed for someone that is technologically aware and responsible, this trade-off provides safety against unintended assistant mistakes without requiring heavyweight virtual machines.

---
## Additional Security Measures
Aside from ONI, the following measures are present to enhance security:
- **Output Sanitization**: Tool and subagent outputs pass through delimiter sanitization to prevent untrusted content from injecting prompt directives into dependent DAG nodes.
- **Approved Write Paths**: ONI restricts file writes strictly to `tools/`, `tests/`, `memories/`, and `logs/`. Critical files like `.env` and root directory configurations are write-protected so generated code cannot overwrite API keys or configurations.
- **Background Task Trust Isolation**: Memory workers and reflection loops are hardcoded to `whitelist_only`. Even if an interactive session runs in `yolo` mode, background tasks never inherit elevated privileges.

---

## Observability: Metrics and Dashboard

<!-- TODO: Add Dashboard Pic here so the significance is properly communicated -->

The more a system grows, the more it becomes necessary to inspect its performance. For MAVIS, the following metrics are tracked:
1. **Latency**: In context of both planning and execution (Mean, Median, and Max are tracked to understand latency distribution).
2. **Failure Rate**: Node failures, tool build errors, and subagent turn-cap limits.
3. **Token Usage**: Prompt and completion token economics broken down across subsystems.
4. **Caching Metrics**: Hit rate over eligible queries, bypass frequency, and estimated tokens saved.

MAVIS has its own logging functionality that appends timestamped logs to component-specific CSVs in `data/metrics/` during runtime with zero database locking overhead. These log CSVs are then used by the Streamlit dashboard (accessed through `/dashboard`) and the in-terminal `/metrics` command for calculating and displaying metrics. 

Moreover, the dashboard allows the user to inspect metrics turn-by-turn by utilizing a lazy join on `turn_id` recorded in the CSVs, thereby offering a comprehensive view of the system's execution lifecycle.


---

*In Part 3, I will discuss the memory subsystem: three tiers of retention, discrete emotion labeling, and topic-based knowledge graphs.*
