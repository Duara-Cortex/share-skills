---
fixtures: ../../../../evals/fixtures/cluster-memory.json
rubric: ../../../../evals/rubrics/cluster-memory.md
---

# Cluster Memory Calibration Guide

This calibration guide instructs evaluators executing self-scoring evaluations for the **Cluster Memory Skill** under the `/eval` command protocol runner.

---

## 🎯 Evaluation Overview

When evaluating the `cluster-memory` skill against the central test fixtures in `evals/fixtures/cluster-memory.json` using the rubric in `evals/rubrics/cluster-memory.md`, evaluate each criterion as **PASS (1)**, **FAIL (0)**, or **N/A**.

The denominator includes only applicable criteria for each fixture:
$$\text{Score (\%)} = \frac{\sum \text{PASS}}{\sum \text{Applicable Criteria}} \times 100$$

---

## 🔍 Criterion-by-Criterion Calibration

### 1. Sensory Signal Extraction (Stage 1 — `filter`)

- **1.1 Correct gating invocation**:
  - **PASS**: The agent issues `sekha-cluster-tool filter` with `--text` and `--directive` (and optional `--threshold`). It reads the `chunks` array from `stdout` and forwards only salient chunks to subsequent stages, reporting `reduction_rate` and `noise_discarded`.
  - **FAIL**: The agent invents an unsupported CLI tool, skips filtering when processing a raw multi-event stream, or fails to extract the salient chunk.
- **1.2 No fabricated signal**:
  - **PASS**: Only chunks present in the response payload are carried forward. If `salient_chunks` is 0, the agent explicitly concludes that no signal exceeded the threshold.
  - **FAIL**: The agent carries discarded noise forward or fabricates salient alerts not returned by Node 3.

---

### 2. Associative Recall Grounding (Stage 2 — `recall`)

- **2.1 Correct recall invocation**:
  - **PASS**: The agent queries Node 1 via `sekha-cluster-tool recall` using a concise `--query` distilled from the salient sensory chunk, supplying a bounded `--top-k`.
  - **FAIL**: The query is empty, ungrounded, or issued to the incorrect endpoint.
- **2.2 Score-ranked, un-padded grounding**:
  - **PASS**: Grounding references are ordered by composite `score` (or vector `sim_score`), relational `edges` are accurately cited, and no unretrieved entities are mentioned.
  - **FAIL**: Grounding entities are reordered arbitrarily, edge relationships are inverted, or external policies are fabricated.

---

### 3. Scratchpad Reasoning (Stage 3 — `deliberate`)

- **3.1 Correct deliberation invocation**:
  - **PASS**: The agent executes `sekha-cluster-tool deliberate` threading the explicit `--task`, the sensory observation (`--input`), and distilled grounding facts (`--context`). It reports the scratchpad's deliberated `thought` and `proposed_action`.
  - **FAIL**: Omission of grounding context, or failure to extract the `thought` and `proposed_action`.
- **3.2 Honours completion state**:
  - **PASS**: When `is_complete` is `true`, the deliberation concludes; when `false`, the agent recognises that further steps are required.
  - **FAIL**: Concluding prematurely on an incomplete step or ignoring an incomplete trajectory flag.

---

### 4. Consolidation Integrity (Stage 4 — `consolidate`)

- **4.1 Correct consolidation invocation**:
  - **PASS**: The agent invokes `sekha-cluster-tool consolidate` with `--goal`, `--outcome`, and `--session-id`, confirming the returned `trace_id` and terminal `status` (`consolidated`).
  - **FAIL**: Consolidation is omitted after completing an action, or invalid outcome states are supplied.
- **4.2 No premature or fabricated consolidation**:
  - **PASS**: Consolidation is only committed once deliberation concludes. Graph statistics (`nodes_updated`, `edges_reinforced`, `entities_extracted`) match the response receipt exactly.
  - **FAIL**: Consolidation is triggered on incomplete reasoning without recording failure, or receipt counts are fabricated.

---

### 5. Telemetry & Latency Budgets

- **5.1 Surfaces telemetry**:
  - **PASS**: The agent extracts and reports latency figures (`latency_ms`, `query_latency_ms`, `total_latency_ms` / `total_duration_ms`). For `orchestrate`, it reads the per-stage `stages[]` execution status.
  - **FAIL**: Telemetry metrics are completely omitted when explicitly queried.
- **5.2 Respects hop budgets**:
  - **PASS**: The agent evaluates latency against the sub-second per-hop SLA ($< 1000\text{ ms}$ per hop) and flags abnormal delays or timeouts.
  - **FAIL**: Excessive latency breaches ($> 3000\text{ ms}$ total or $> 1000\text{ ms}$ single-hop) are ignored without comment.

---

### 6. Contract & Style Discipline

- **6.1 JSON contract adherence**:
  - **PASS**: The agent consumes structured JSON exclusively from `stdout`, distinguishing it from diagnostic step logs emitted on `stderr`.
  - **FAIL**: The agent fails to parse JSON or attempts to parse freeform text diagnostics.
- **6.2 Frozen keys**:
  - **PASS**: Exact JSON property keys (`salient_chunks`, `reduction_rate`, `sim_score`, `proposed_action`, `is_complete`, `trace_id`, `latency_ms`) are preserved without renaming, re-casing, or anglicisation.
  - **FAIL**: Modifying JSON keys (e.g. changing `is_complete` to `is_completed` or `sim_score` to `similarity_score`).
- **6.3 British English prose**:
  - **PASS**: All surrounding commentary, summaries, and explanations use pure British English (`en_GB`) spelling (e.g. *initialise*, *serialise*, *optimise*, *neighbour*, *behaviour*, *prioritise*).
  - **FAIL**: Any instance of American English spelling in generated text.

---

### 7. Fallback & Degradation Protocol

- **7.1 Configuration gating**:
  - **PASS**: If an endpoint is unconfigured or blank, the agent refuses to guess an IP address and guides the operator to run `sekha-cluster-tool env init` and configure `.env`.
  - **FAIL**: Guessing default private cluster IP addresses (e.g. `192.168.x.x`) when the configuration is blank.
- **7.2 Graceful degradation**:
  - **PASS**: On an unreachable node or timeout, the agent gracefully logs the failure from `stages[]`, executes the fallback heuristic (e.g. bypassing Stage 1 with unit salience), and avoids crashing the runner.
  - **FAIL**: Unhandled exceptions, silent failure swallowing, or fabricating results for an offline node.

---

## 🚀 Execution Instructions

To evaluate this skill:
```bash
/eval antigravity/cluster-memory
```
Ensure all 6 fixtures (`CM-01` to `CM-06`) pass with an aggregate score $\ge 90\%$ (Grade: **Excellent**).
