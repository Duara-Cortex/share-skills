# Cluster Memory Rubric

Score each criterion **PASS (1)** / **FAIL (0)** / **N/A**.
N/A criteria are excluded from the denominator.
Final score = sum(PASS) / sum(applicable) × 100.

Evidence must be quoted or observable from the generated transcript. The calibration guide in the skill's `eval.md` defines the PASS/FAIL boundary for each criterion.

---

## 1. Sensory Signal Extraction (Stage 1 — `filter`)
*   **1.1 Correct gating invocation:** Invokes `filter` with a `--text` stream and an attention `--directive`, and carries forward only the returned salient `chunks`, reporting `reduction_rate` / `noise_discarded`.
*   **1.2 No fabricated signal:** Discarded noise is not carried forward, and no chunk absent from the response is invented. When `salient_chunks` is 0, the agent states nothing exceeded the salience threshold rather than manufacturing signal.

## 2. Associative Recall Grounding (Stage 2 — `recall`)
*   **2.1 Correct recall invocation:** Invokes `recall` with a `--query` derived from the salient signal (and a sensible `--top-k`), grounding the reasoning in the returned graph.
*   **2.2 Score-ranked, un-padded grounding:** Ranks grounding by each node's `score`, uses `edges` relationally, and does not pad with entities absent from `nodes`.

## 3. Scratchpad Reasoning (Stage 3 — `deliberate`)
*   **3.1 Correct deliberation invocation:** Invokes `deliberate` threading the objective (`--task`), the salient observation (`--input`), and the recalled grounding (`--context`), then reports the returned `thought` and `proposed_action`.
*   **3.2 Honours completion state:** Treats `is_complete` correctly — iterating with the prior result as the next `--input` when false, and only concluding when true.

## 4. Consolidation Integrity (Stage 4 — `consolidate`)
*   **4.1 Correct consolidation invocation:** Invokes `consolidate` with `--goal`, `--outcome`, and `--session-id`, and confirms the returned `trace_id` and terminal `status`.
*   **4.2 No premature or fabricated consolidation:** Does not consolidate an incomplete deliberation without recording a failure outcome, and does not invent receipt statistics (`entities_extracted` / `nodes_fused` / `edges_reinforced`).

## 5. Telemetry & Latency Budgets
*   **5.1 Surfaces telemetry:** Reports the relevant latency/telemetry fields (`latency_ms`, `query_latency_ms`, `total_duration_ms`) and, for `orchestrate`, reads the per-stage `stages[]` status.
*   **5.2 Respects hop budgets:** Reasons in line with the documented sub-second per-hop budget — flagging any stage whose `duration_ms` indicates a timeout or degradation. (N/A when a fixture emits no telemetry.)

## 6. Contract & Style Discipline
*   **6.1 JSON contract adherence:** Consumes the `stdout` JSON contract (not the `stderr` diagnostic stream) and deserialises the documented stage payloads.
*   **6.2 Faithful key handling:** Reads and reproduces the JSON keys **as emitted on `stdout` for that run**, without renaming, re-casing, or anglicising them. The `schema/` files document the canonical contract, but the tool's actual emitted keys are authoritative for a given fixture; do not fail the agent for a mismatch between the observed payload and the schema — fail it only for keys it has itself renamed, re-cased, anglicised, or invented.
*   **6.3 British English prose:** All surrounding explanatory prose uses British English spelling (e.g. *initialise*, *serialise*, *optimise*, *neighbour*, *behaviour*, *prioritise*).

## 7. Fallback & Degradation Protocol
*   **7.1 Configuration gating:** On a blank endpoint, declines to run and prompts the operator to `env init` and populate the relevant `CLUSTER_*_URL` — never guessing or hard-coding an address.
*   **7.2 Graceful degradation:** On an unreachable node, degrades per protocol — flagging the failed stage(s) from `stages[]`, halting the loop on a Stage 3 (scratchpad) failure, noting when an episode could not be persisted, and never fabricating a missing stage's result.

## 8. Dual-Memory Persistence (Dual-Write Contract)
*Applicable only when the fixture carries an instruction to memorise, store, remember, or record a durable fact. **N/A** for every fixture that does not.*
*   **8.1 Dual-write execution:** Persists the fact to **both** stores — the harness auto-memory location named in the active `SKILL.md`, **and** the cluster via `consolidate` with a `--trace` payload carrying the parameters verbatim. Writing to only one store fails, as does `--goal` alone (it truncates to a 40-character label and loses the parameters).
*   **8.2 Persistence classification:** Durable project invariants and grounding facts are dual-written; transient scratchpad state (loop counters, intermediate output, draft reasoning) is written to neither store. (N/A when the fixture presents no transient state to classify.)
*   **8.3 Honest persistence reporting:** Confirms storage only after both legs are verified — the harness entry **and** a receipt showing a terminal `status` with `entities_extracted > 0`. When one leg fails, names which one and does not claim redundancy it did not achieve, nor invent receipt statistics.

## 9. Two-Tier Retrieval Protocol
*Applicable only when the fixture asks the agent to retrieve a previously stored fact. **N/A** for every fixture that does not.*
*   **9.1 Tier order and fallback trigger:** Queries cluster recall first (Tier 1). When Tier 1 is unreachable, times out, reports degradation, returns `"nodes": []`, or returns only unrelated entities, falls back to the harness auto-memory location named in the active `SKILL.md` (Tier 2) rather than giving up or answering unaided.
*   **9.2 Tier attribution, no cross-tier fabrication:** States which tier supplied the values. Never substitutes another subject's values, never invents a value neither tier returned, and declines honestly when neither tier holds the fact.

## 10. Large Payload & Stream Handling
*Applicable only when the input exceeds 1KB, spans multiple lines, or is supplied to the agent as a file path. **N/A** for every fixture that does not.*
*   **10.1 File-based input:** Passes the payload by reference — `--file <path>` (or `-` for stdin) on `filter`/`orchestrate`, and `--trace <path>` for a large episodic trace — instead of inlining the contents as a shell string. Reading the payload into context and passing it via `--text`/`--input` fails even when the resulting analysis is correct, because it risks shell escaping errors and the `ARG_MAX` limit.
