---
layout: default
title: "MAVIS Series"
nav_order: 2
has_children: true
permalink: /mavis/
---

# MAVIS Architecture Series
{: .fs-9 }

An engineering deep-dive series on designing and building **MAVIS** which is a self-modifying personal assistant that can make its own tools and subagents. 

These posts discuss the technical design and choices of various components by first presenting the design problem and then discussing the architectural design to solve the problem.

---

## High-Level Architecture

```
User Input
    │
    ▼
[Interpreter] ──► Compiles intent into a validated feedforward DAG
    │
    ▼
[Executor] ─────► Executes tool nodes in topological order
    │
    ├─► [Dynamic Tool Synth] (AST scan -> ToolTester -> subprocess)
    ├─► [Multi-Tier Memory]  (Working -> Episodic -> Semantic)
    └─► [ONI Policy Gate]   (Whitelist -> Ask -> Blacklist)
    │
    ▼
[Answerer] ─────► Synthesizes final response for user
```

---

## Series Outline

Below are all articles in the series:
