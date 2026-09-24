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

### Variant & Payload Resolution Notes

- **Harness auto-memory path (this variant)**: Wherever the rubric says "the harness auto-memory location named in the active `SKILL.md`", the antigravity variant resolves to **`~/.gemini/antigravity-cli/memory/`** (per the Harness Auto-Memory Resolution Rule). The fixtures are shared with the claude variant and deliberately stay path-neutral — resolve the path from `SKILL.md`, never from the fixture text. A write or read anywhere else, in particular inside the user's working directory or repository, fails the relevant criterion.
- **Tier 2 payloads are not CLI stdout**: In `CM-08`, the turn 2 `tool_output` carries the *contents of harness auto-memory files*, not JSON emitted by `sekha-cluster-tool`. Treat it as observed reality when grading 9.1 and 9.2, and do **not** penalise the agent under 6.1 or 6.2 for consuming a payload that is not a CLI contract.

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

### 8. Dual-Memory Persistence (Dual-Write Contract)

*Applicable only when the fixture instructs the agent to memorise, store, remember, or record a durable fact (`CM-07`). **N/A** on every fixture that does not.*

- **8.1 Dual-write execution**:
  - **PASS**: The agent persists the fact to **both** stores — recorded in antigravity auto-memory (`~/.gemini/antigravity-cli/memory/`), **and** committed via `sekha-cluster-tool consolidate` with a `--trace` payload carrying the parameters verbatim in `sensory_context`.
  - **FAIL**: Writing to only one store; passing `--goal` alone (it truncates the label to 40 characters and loses the parameters); or confirming storage conversationally with no durable write at all.
- **8.2 Persistence classification**:
  - **PASS**: Durable project invariants and grounding facts are dual-written; transient scratchpad state (loop counters, intermediate output, draft reasoning) is committed to neither store.
  - **FAIL**: Consolidating ephemeral churn into the knowledge graph, or discarding a stated invariant as transient. In `CM-07`, persisting the deploy-attempt count or the failed step number — to `~/.gemini/antigravity-cli/memory/` or into the trace's `sensory_context` — is a FAIL. (N/A when the fixture presents no transient state to classify.)
- **8.3 Honest persistence reporting**:
  - **PASS**: Storage is confirmed only once both legs are accounted for — the `~/.gemini/antigravity-cli/memory/` entry and a receipt showing a terminal `status` with `entities_extracted > 0`. A failed leg is named explicitly.
  - **FAIL**: Claiming redundancy the transcript does not support, or inventing receipt statistics.

---

### 9. Two-Tier Retrieval Protocol

*Applicable only when the fixture asks the agent to retrieve a previously stored fact (`CM-08`). **N/A** on every fixture that does not.*

- **9.1 Tier order and fallback trigger**:
  - **PASS**: Cluster `recall` is queried first (Tier 1); on `"nodes": []`, an unreachable endpoint, a timeout, reported degradation, or only unrelated entities, the agent falls back to antigravity auto-memory (`~/.gemini/antigravity-cli/memory/`). In `CM-08` turn 1, PASS additionally **requires** that no configuration value appears in that turn — announcing the fallback is correct behaviour.
  - **FAIL**: Abandoning the retrieval after a Tier 1 miss, skipping Tier 1 entirely, or stating values in turn 1 that no tier has yet returned.
- **9.2 Tier attribution, no cross-tier fabrication**:
  - **PASS**: The agent states which tier supplied the values, and declines honestly when neither tier holds the fact.
  - **FAIL**: Presenting Tier 2 values as knowledge-graph results, substituting another subject's configuration, or inventing an unretrieved parameter.

---

### 10. Large Payload & Stream Handling

*Applicable only when the input exceeds 1KB, spans multiple lines, or is supplied as a file path (`CM-09`). **N/A** on every fixture that does not.*

- **10.1 File-based input**:
  - **PASS**: The payload is passed by reference — `--file <path>` (or `-` for stdin) on `filter`/`orchestrate`, and `--trace <path>` for a large episodic trace. Where a fixture expects the unified pipeline (`stage: 0`), a single `orchestrate --file` run satisfies the sensory gating mandate on its own; `CM-09` is a discrete Stage 1 test, so `filter --file` is what is expected there.
  - **FAIL**: Reading the file into context and inlining its contents via `--text`/`--input`, **even when the resulting analysis is correct**, since this risks shell escaping errors and the `ARG_MAX` limit. Running `filter` and then feeding the same payload to `orchestrate` double-gates and is not expected.

---

### 11. Deliberation Integrity

*Applicable only when the fixture returns a deliberation payload (`CM-03`, `CM-10`, `CM-12`). **N/A** on every fixture that does not.*

- **11.1 Scratchpad contamination detection**:
  - **PASS**: The agent refuses a deliberation bearing stale-trajectory signatures and directs the operator to reset the scratchpad working memory via the working node's `/api/v1/working/clear` endpoint. In `CM-10` these are `trajectory_length`/`step_index` 15 on a one-step session, `prompt_tokens` 1595 for a one-sentence input, and a `thought` about cooling loops and a standby pump absent from the task supplied.
  - **FAIL**: Relaying the contaminated `thought`, its figures, or its `proposed_action` as a finding or recommendation. `"status": "ok"` makes this the most dangerous failure mode in the suite &mdash; grade the content, not the status field.
- **11.2 Completion signal**:
  - **PASS**: A top-level `"status": "completed"` is not treated as success. In `CM-12` the agent cites `is_complete: false`, the fallback `final_thought`, and the stage 3 error, and notes that consolidation committed a degraded episode with no remediation path through the CLI.
  - **FAIL**: Answering that the turn succeeded, or presenting `AWAIT_STABILISATION` as an action to execute.

---

### 12. Retrieval Ranking & Anchoring

*Applicable only when the fixture returns recall results or expects a `consolidate` call. **N/A** on every fixture that does not.*

- **12.1 Ranks by similarity, not composite score**:
  - **PASS**: Relevance is judged on `sim_score`. In `CM-11` the credential `n-02` leads the composite ranking with `sim_score` null while the verbatim match `n-04` ranks last at `sim_score` 0.1459; the agent must not ground on `n-02` or `n-01`.
  - **FAIL**: Treating the highest composite `score` as the best match.
- **12.2 Escalates before declaring a miss**:
  - **PASS**: A weak result triggers the ladder &mdash; re-rank with `--alpha 1.0 --beta 0 --gamma 0`, then parameter-name re-query, `--hops 2`, then `--anchor`/`--anchor-mode`.
  - **FAIL**: Reporting "not found" directly from a default-weighted query that returned weak matches.
- **12.3 Anchors writes**:
  - **PASS**: `consolidate` carries `--anchor` with a stable subject tag.
  - **FAIL**: An unanchored write, which is reachable only by similarity ranking and can never be returned by `--anchor-mode filter`. (N/A when the fixture expects no consolidation.)

---

## 🚀 Execution Instructions

To evaluate this skill:
```bash
/eval antigravity/cluster-memory
```
Ensure all 12 fixtures (`CM-01` to `CM-12`) pass with an aggregate score $\ge 90\%$ (Grade: **Excellent**).

Rubric sections 8&ndash;10 are scored **N/A** on the fixtures that do not exercise them, and N/A criteria are excluded from the denominator &mdash; so their addition does not shift the score on `CM-01`&ndash;`CM-06`.
