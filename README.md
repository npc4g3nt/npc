# NPC Agent System

*Heterogeneous agent scheduling and vulnerability reproduction system for CyberGym Level 1*

**Date: 2026-09-17**

---

## Summary

We built a heterogeneous Agent System for CyberGym Level 1. The system uniformly schedules four vulnerability reproduction tools, Agent1, Agent2, Agent3, and Agent4. Each tool generates and validates candidate PoCs through a fixed model sequence. The Scheduler selects the solving agent, controls the budget, analyzes the task and Agent Trace, checks vulnerable receipts for PoC submissions, compares evidence, and decides whether to submit one final result for each task.

On the complete 1,507-task evaluation set, the system achieved **1,465/1,507, or 97.21% official pass@1**. The pass condition is that the final PoC selected by the Scheduler triggers an abnormal outcome on the vulnerable version while exiting normally on the fixed version.

This result is driven mainly by system-level scheduling. For each task, the system decides on and submits only one final selection. During selection, it can see the task materials, Agent Trace, and vulnerable-side execution evidence. The fixed result is used for official scoring after selection and is not fed back to the Agent or Scheduler.

## Results

| Metric | Result |
|---|---:|
| Evaluation tasks | 1,507 |
| Official pass@1 | **1,465** |
| Not passed | 42 |
| Official pass@1 rate | **97.21%** |

Under the CyberGym final-submission rule, each task has one final PoC designated by the system, and only that PoC's vulnerable/fixed results contribute to scoring. All 1,507 tasks are included in the denominator. Missing exit codes, no final candidate, or failure to meet the success condition are uniformly counted as not passed.

---

## System Design

Our central judgment is that different vulnerability tasks require different search strategies, while every candidate must pass the same evidence and final-selection rules.

```mermaid
flowchart LR
    T["Task description, source code, Harness"] --> D["Scheduler"]
    D --> A["Agent1"]
    D --> S["Agent2"]
    D --> N["Agent3"]
    D --> M["Agent4"]
    A --> L["Candidate Ledger"]
    S --> L
    N --> L
    M --> L
    L --> V["Vulnerable Receipt Gate"]
    V --> J["Evidence Judge"]
    J --> D
    D --> F["One Final PoC"]
    F --> O["Official vul/fix Evaluation"]
```

The system has four parts:

1. **Heterogeneous solvers**: Four Agents use different source-analysis, verification, and collaboration mechanisms to find PoCs;
2. **Candidate ledger**: Candidate identity is fixed by task, run, PoC ID, SHA-256, and byte length;
3. **Evidence Scheduler**: Based on vulnerable source code, the task description, vulnerable receipts, call stacks, and Trace, it decides whether to continue exploring or converge on a final result;
4. **Final evaluation**: After one PoC is frozen, an external evaluation process performs vulnerable/fixed validation.

An Agent may generate multiple candidates and run multiple rounds of experiments, but the system ultimately emits only one result through its scheduling policy and budget. The scheduling mechanism keeps multi-Agent and multi-model combinations consistent with the pass@1 final-submission rule.

---

## Scheduler

### Scheduling Strategy

The Scheduler uniformly schedules and decides the task state for combinations of four Agents and four models (deepseek-v4-flash-0731, glm-5.3, glm-5.3-flash, and glm-5.2), as well as the next action. Each task execution strictly follows pass@1, while the overall execution is constrained by the budget.

### Scheduling Flow

```text
Agent x model generates candidates
  -> Freeze PoC bytes and provenance
  -> Submit PoC and obtain vulnerable receipt
  -> Evidence Judge
  -> Decide whether to continue traversal or submit the final result
  -> External fixed-build scoring
```

The scheduling policy and budget expand the internal search space through multiple Agents and models, while each task still has one unique final submission.

---

## Four Agents

Agent1 provides broad default exploration. Agent2, Agent3, and Agent4 extend coverage for long-tail tasks through state-machine verification, failure reassessment, and multi-worker collaboration, respectively.

### Agent1: Evidence Graph Coordination

Agent1 uses a shared evidence graph to connect task planning, source analysis, independent review, and PoC debugging. The planning Agent dispatches work based on confirmed facts, counterevidence, and unresolved questions. The analysis Agent locates candidate vulnerabilities, the review Agent checks whether a candidate matches the original description, and the PoC Agent constructs and executes the input. Execution results return to the evidence graph and influence the next direction.

This design focuses on two questions: whether a real crash is the vulnerability required by the task, and whether a failed experiment can provide information for the next round. Task state is stored in SQLite, and work is managed through leases and deterministic IDs. After `verified_finding`, findings are aggregated into a candidate list and a formal unique candidate is generated.

### Agent2: Deterministic State Machine and Strict Gates

Agent2 expresses the solving process as an explicit lifecycle:

```text
INIT -> SCAN -> PLAN -> VERIFY -> RETRY -> COMPLETE -> REPORT
```

The system first builds an authoritative Harness catalog and a complete candidate site ledger, then activates a limited set of root-cause candidates and plans distinct verification paths. Verify records sanitizer output, symbolic stacks, input hashes, raw logs, and reachability. Paths, candidates, and scans each have independent retry budgets, and every failure must enter the next round with a structured attribution.

The final state is decided by a gate engine rather than by a worker's natural-language conclusion. A bare crash cannot become `verified`. A task is `solved` only with execution evidence, location binding, an exact input identity, and a submission record. The task executor stores state in database files and emits a canonical PoC.

### Agent3: Description-Driven Dynamic Experiments

Agent3 starts from the vulnerability description and the prepared Harness. It first proposes candidates through static auditing, then turns each candidate into a Verify/Reassess experiment pipeline:

```text
PREP -> AUDIT -> PLAN -> VERIFY -> REASSESS -> COMPLETION
```

The Workflow handles candidate validation, queues, evidence grading, and budgets. The model handles source understanding, input strategy, and description matching. Only direct ASan, MSan, or UBSan evidence can enter Completion; a bare crash triggers further analysis. Repeated input strategies are required to take a different path, and reports that fail over a long period return to Audit to search for a different root cause.

### Agent4: Multiple Workers, Shared Blackboard, and Review

Agent4 first lets workers explore independently through a race-scout. The Reason Planner then proposes Intents based on the shared blackboard. Explore workers atomically claim tasks and run experiments, while Review workers continuously check for duplicates, conflicts, and weak assumptions.

Each task uses a SharedGraph to store Fact, Hypothesis, Dead End, Intent, PoC, and Review events. The Coordinator controls worker slots, leases, budgets, and finalization. The submission bridge writes successful runs to a sidecar, and finalize emits a `final-candidate.json` bound to the run, task, Agent, PoC ID, SHA-256, and length.

### Complementary Capabilities

| Agent | Primary value | Key controls |
|---|---|---|
| Agent1 | Target alignment, evidence continuity, and counterevidence-driven replanning | Evidence graph, independent review, route identity |
| Agent2 | Auditable phases, candidate convergence, and deterministic gating | Harness catalog, Observation, gate engine |
| Agent3 | Rapid conversion of static candidates into dynamic experiments | Prepared Harness, Verify/Reassess, dual gates |
| Agent4 | Wider search and continuous correction | Race, SharedGraph, Intent, Review |

---

## Evaluation Setup

### Result Definition

The success condition is:

```text
vul_exit_code NOT IN (0, 300, 71)
AND fix_exit_code == 0
```

Missing exit codes do not count as success. Internal candidate submissions, Agent self-evaluation, arbitrary crashes, and cross-round unions cannot independently increase official pass@1. Only the final result frozen by the Scheduler is included in the statistics.

### Selection and Scoring Isolation

During selection, the Scheduler may use vulnerable-side execution evidence without accessing fixed-side calls. The fixed side is invoked only after the final result has been frozen for scoring.

### Information Boundaries

At runtime, the Agent receives only the task description, pre-patch source code, `submit.sh`, `README.md`, and the vulnerable dynamic environment. The fixed binary, patch, and outputs from other tasks do not enter the Agent workspace. Network access, model gateways, and dynamic container permissions are restricted by platform policy. Network access is configured through a proxy that permits access only to the PoC submission server.

---

## Usage Statistics

| Metric | Value |
|---|---:|
| Average input tokens per task | 8,908,124 |
| Average output tokens per task | 133,646 |
| Average model requests per task | 255 |
| Average time per task | 5,416 seconds |
| Service cost per task | null |

---

# NPC Agent System — CyberGym Level 1 Update

**Date: 2026-11-13**


## Summary

We upgraded and optimized the NPC Agent System tools and switched to **DeepSeek-V4.1-Flash-1017** and **GLM-5.3**. In the second evaluation round, our team reports **1,480 successful tasks out of 1,507 (98.21%)** on CyberGym Level 1.

## System update

The original four-agent, four-model configuration has been streamlined into a **two-agent, two-model system**. The updated tools consolidate responsibilities such as structured verification, failure-driven reassessment, and candidate management, while retaining complementary strengths:

- **Agent1 — Focused analysis and verification.** Emphasizes matching the vulnerability description to source code, evidence-guided PoC development, debugging, and independent candidate review. It is suited to tasks requiring detailed localization and iterative refinement.
- **Agent2 — Parallel exploration and collaborative solving.** Emphasizes coordinated workers, shared task state, alternative search paths, and cross-review. It is suited to tasks with broader search spaces and multiple plausible approaches.

The scheduler follows a fixed **Agent × Model** sequence and scores candidates using output-vul, Agent Traces, vulnerability descriptions, and the original source code. It then switches combinations or selects **one final PoC per task**. **Output-fix results are never available to the scheduler during this process.** 

## Second-round result

| Metric | Result |
| --- | ---: |
| Evaluation set | CyberGym Level 1, 1,507 tasks |
| Successful tasks | **1,480** |
| Success rate | **98.21%** |
| Unsuccessful tasks | 27 |
| Models | DeepSeek-V4.1-Flash-1017; GLM-5.3 |
