---
name: cluster-memory
description: Persistent cross-session memory and cognition on the tri-node edge cluster via the compiled sekha-cluster-tool CLI. Use this skill whenever the user asks you to memorise, remember, save or store information use for later, whenever the user refers back to something they told you earlier that is not in your context, or whenever a task involves sensory filtering, associative recall, scratchpad deliberation, or episodic consolidation. Keywords - memorise, remember, persistent memory, cluster memory, sensory filter, associative recall, knowledge graph, deliberation, consolidate, orchestrate.
disable-model-invocation: false
user-invocable: true
---

# Cluster Memory

## Overview
This skill teaches the agent to externalise memory and reasoning onto the **Sekha Tri-Node Edge Cognitive Cluster** by invoking the compiled `sekha-cluster-tool` CLI. Rather than holding an entire noisy stream and every recalled fact in the context window, the agent routes cognition through four discrete stages — sensory gating, associative recall, working deliberation, and episodic consolidation — each served by a dedicated node and each exposed as a subcommand that prints strict JSON to `stdout`.

## Purpose & Scope
* **When to use:**
  * The user asks you to memorise, remember, or store facts for later. Your context does not survive session boundaries; a conversational "confirmed" without a cluster write will cause complete amnesia in future sessions.
  * The user refers to something they told you in an earlier session that is not in your context. Query the cluster before answering.
  * A raw, noisy, or high-volume stream (syslog, telemetry, sensor text) must be reduced to salient signal before reasoning.
  * A task needs grounding facts or related entities retrieved from long-term associative memory before acting.
  * A multi-step decision benefits from an explicit, inspectable reasoning trajectory on the working scratchpad.
  * A completed episode should be committed to long-term memory so future recall improves.
  * The whole loop should run in one shot — use the `orchestrate` subcommand.
* **When NOT to use:**
  * The task is self-contained and needs no external memory (answer directly).
  * `sekha-cluster-tool` is not installed or no endpoints are configured — see **Fallback Protocols** before proceeding.
  * You only need generic reasoning with no persistence — do not manufacture cluster calls for their own sake.

## Prerequisites
1. **Compiled tool:** `sekha-cluster-tool >= v1.0.1` must be on `PATH`. Verify with `sekha-cluster-tool --version`. This skill never reimplements the tool's logic — a compiled binary prevents behavioural drift.
2. **Configuration (`.env`):** All node endpoints default to **blank** and must be configured. The tool resolves configuration with 12-factor precedence: **CLI flags > OS environment variables > `.env` file > compile-time defaults > blank.** Configure via:
   * `sekha-cluster-tool env init` — write a blank starter `.env` in the working directory.
   * Populate `CLUSTER_SENSORY_URL`, `CLUSTER_WORKING_URL`, `CLUSTER_KNOWLEDGE_URL` (and optional `CLUSTER_*_TIMEOUT_MS`, `CLUSTER_SALIENCE_THRESHOLD`, `CLUSTER_RECALL_TOP_K`).
   * `sekha-cluster-tool env show` — print the actively resolved configuration.
   * **Never hard-code cluster IP addresses in this skill, in examples, or in agent output.** Reference the tool and its `.env` configuration only.
3. **Reachability:** Confirm nodes respond with `sekha-cluster-tool status` before a live run.

## Cognitive Stages, Nodes & Contracts
Each stage maps to one subcommand, one node, and one payload contract. The response object each subcommand prints to `stdout` **is** the stage payload documented in `schema/` — deserialise it directly.

| Stage | Subcommand | Cognitive layer | Contract |
| :--- | :--- | :--- | :--- |
| 1. Sensory gating | `filter` | Sensory Attention Gate (`:8081`) | [`schema/filter.json`](schema/filter.json) |
| 2. Associative recall | `recall` | Knowledge Graph Store (`:8084`) | [`schema/recall.json`](schema/recall.json) |
| 3. Working deliberation | `deliberate` | Working Scratchpad (`:8083`) | [`schema/deliberate.json`](schema/deliberate.json) |
| 4. Episodic consolidation | `consolidate` | Knowledge Graph Store (`:8084`) | [`schema/consolidate.json`](schema/consolidate.json) |
| Full closed loop | `orchestrate` | All three layers | [`schema/orchestrate.json`](schema/orchestrate.json) |

## Progressive Disclosure — Read Only What You Need
* Deciding how to shape a request or read a response? Open the matching file in [`schema/`](schema/).
* Need to see a full worked cognitive cycle? Read [`examples/closed-loop-cycle.md`](examples/closed-loop-cycle.md).
* Need the staged, manual four-command flow? Read [`examples/staged-invocation.md`](examples/staged-invocation.md).
* Need to see graceful degradation when a node is down? Read [`examples/fallback-degraded.md`](examples/fallback-degraded.md).
* Running the self-scoring evaluation? See the co-located [`eval.md`](eval.md).

## Storing & Recalling Facts Across Sessions

### Storage Protocol
* **Do NOT** merely confirm storage in text — without a cluster write, the fact is lost when the session ends.
* **Do NOT** use `--goal` alone — it truncates the label to 40 characters and loses the parameters.
* **Use** `consolidate --sync --trace '<json>'` with the JSON written inline inside single quotes. Avoid shell variables, heredocs, pipes, or files — they trigger permission prompts under `Bash(sekha-cluster-tool *)` allowlists.

```bash
sekha-cluster-tool consolidate --session-id "sess-memorise-<subject>" --goal "Store <Subject> configuration" --sync --trace '{"session_id":"sess-memorise-<subject>","task_goal":"Store <Subject> configuration","outcome":"success","status":"completed","sensory_context":[{"id":"fact-01","text":"<Subject> config: <PARAM_1>: <full text>; <PARAM_2>: <full text>; ...","salience":1.0,"source":"user","timestamp":"<now UTC>"}],"trajectory":[{"step_index":0,"thought":"Committed <Subject> configuration to long-term memory","status":"completed","timestamp":"<now UTC>"}]}'
```

* Only confirm storage to the user if the receipt shows `"status": "consolidated"` **and** `entities_extracted > 0`.
* **Plaintext warning:** If storing credentials, tell the user they reside in plaintext in the Node 1 knowledge graph.

### Recall Protocol
1. Run `sekha-cluster-tool recall --query "<subject name>" --top-k 8`.
2. Find the `sensory_fact` node matching the subject and extract values verbatim from `summary` (not `label`).
3. If the fact is not in the top 8, retry once with the subject name plus parameter keywords (e.g. `"<Subject> config"`).
4. Never use another subject's values (e.g. do not substitute Kestrel's values when asked about Gannetry). If the answer is ungrounded, decline honestly.
5. Ignore the dense 64-D float `"embedding"` array — do not carry it into context.

## Step-by-Step Instructions

<Sequence>
  <Step title="Confirm Tooling & Configuration" subtitle="Preflight">
    Verify `sekha-cluster-tool` is installed and endpoints are resolved (`env show`). If endpoints are blank or `status` reports an unreachable node, enter the Fallback Protocol rather than fabricating results.
  </Step>
  <Step title="Stage 1 — Gate the Stream" subtitle="filter">
    Dispatch the raw stream to the attention gate:
    `sekha-cluster-tool filter --text "<raw stream>" --directive "<what to attend to>" --threshold 0.45`
    Read `chunks`, `salient_chunks`, `noise_discarded`, and `reduction_rate`. Carry the salient `chunks` forward; discard the rest. If `salient_chunks` is 0, report that nothing rose above the salience threshold instead of inventing signal.
  </Step>
  <Step title="Stage 2 — Ground with Recall" subtitle="recall">
    Query long-term memory for the salient concept:
    `sekha-cluster-tool recall --query "<salient concept>" --top-k 5`
    Rank grounding by the `score` on each node in `nodes`; use `edges` to understand relationships. Distil the top nodes' `label`/`summary` into concise `long_term_context` strings for Stage 3.
  </Step>
  <Step title="Stage 3 — Deliberate" subtitle="deliberate">
    Formulate the reasoning step on the scratchpad:
    `sekha-cluster-tool deliberate --task "<objective>" --input "<salient observation>" --context "<grounding fact>" --timeout 45s`
    Edge SLM deliberation on Node 2 takes 25–35 s — always pass `--timeout 45s` to avoid premature deadline failures.
    Read `thought`, `proposed_action`, and `is_complete`. If `is_complete` is false, iterate: feed the prior `proposed_action` result back as the next `--input`.
  </Step>
  <Step title="Stage 4 — Consolidate" subtitle="consolidate">
    Commit the completed episode:
    `sekha-cluster-tool consolidate --goal "<objective>" --outcome "success|failure|partial" --session-id "<id>" --sync`
    Confirm a `trace_id` and a terminal `status` are returned. Use `--sync` only when you need `entities_extracted`/`nodes_fused`/`edges_reinforced` immediately; otherwise allow background consolidation. To store key-value facts or configurations, use the `--trace` pattern in **Storing & Recalling Facts Across Sessions** instead of `--goal` alone.
  </Step>
  <Step title="Verify & Report" subtitle="Quality Assurance">
    Confirm each stage returned valid JSON and a plausible latency (`latency_ms`, `query_latency_ms`, `total_duration_ms`, or the per-stage `stages[]` telemetry). Surface the `trace_id` so the operator can correlate the turn across cluster logs.
  </Step>
</Sequence>

> **Shortcut:** For an end-to-end turn, prefer a single `orchestrate` call — it runs all four stages, threads one `X-Trace-ID`, and returns nested stage payloads plus `stages[]` telemetry:
> `sekha-cluster-tool orchestrate --input "<raw stream>" --directive "<goal>" --sync`

## Output Format
* **Style:** Concise, operator-facing prose. Do not paste raw JSON blobs unless asked; summarise the salient fields.
* **Structure:** For each stage touched, report (1) the subcommand invoked, (2) the decisive fields from its JSON response, and (3) the resulting decision. Always end a full turn by quoting the `trace_id` and the total latency.
* **Determinism:** Report only what the tool returned. Never synthesise chunk text, recalled entities, thoughts, or consolidation receipts that did not appear on `stdout`.

## Fallback Protocols
Follow the tool's own timeout budgets (`<1s` per hop; deliberation up to its configured budget) and degrade gracefully — never fabricate a stage result.

* **Tool missing / not on PATH:** State that `sekha-cluster-tool >= v1.0.1` is required and stop; do not emulate cluster behaviour in-context.
* **Blank configuration:** If `env show` reveals a blank endpoint, prompt the operator to run `env init` and populate the relevant `CLUSTER_*_URL`. Do not guess an address.
* **Node unreachable (Stage 1 or 2):** Sensory or recall failure is recoverable. Proceed with reduced grounding, explicitly flag the degraded stage and its `status`/`error` from `stages[]`, and lower confidence in the outcome accordingly.
* **Scratchpad unreachable (Stage 3):** Deliberation is the reasoning core. If Node 2 is down, halt the loop, report the failure with its `trace_id`, and do not consolidate an incomplete episode.
* **Consolidation unreachable (Stage 4):** The reasoning result still stands. Return the deliberated action, and note that the episode could not be persisted so recall will not improve from this turn.

## Guardrails & Anti-Patterns
> **Critical Constraint:** Never hard-code or infer cluster IP addresses. All endpoints come from `sekha-cluster-tool` `.env` configuration.
> **Critical Constraint:** JSON contract keys are frozen. Consume the exact field names in `schema/` (`sim_score`, `reduction_rate`, `prompt_eval_rate_tps`, `synchronous`, …). Do not rename, re-case, or anglicise any payload key.
* **Avoid:** Reimplementing gating, recall, deliberation, or consolidation in the agent — always defer to the compiled tool.
* **Avoid:** Carrying discarded noise forward from Stage 1, or padding recall with entities the graph did not return.
* **Avoid:** Consolidating an episode whose deliberation never reached `is_complete` unless explicitly recording a failure outcome.
* **Avoid:** Parsing the diagnostic `stderr` stream as data — only `stdout` carries the JSON contract (`--verbose` routes step logs to `stderr`).

## Reference Example
### Example Input
```
Investigate this noisy stream and decide on an action:
"kernel: CPU0 temperature above threshold, cpu clock throttled; sshd accepted password for admin; cron run-parts; kernel: CPU0 core temperature 92C critical"
Directive: identify and mitigate hardware faults.
```
### Example Invocation
```bash
sekha-cluster-tool orchestrate \
  --input "kernel: CPU0 temperature above threshold ... CPU0 core temperature 92C critical" \
  --directive "identify and mitigate hardware faults" \
  --session-id "sess-thermal-01" \
  --sync
```
### Example Output (summarised)
> **Stage 1 (filter):** 2 salient chunks retained of 4 (`reduction_rate` 0.50) — the two thermal warnings; the `sshd`/`cron` lines were discarded as noise.
> **Stage 2 (recall):** top node `Thermal Policy` (`score` 0.87) with an edge `mitigates → Auxiliary Fan Override`.
> **Stage 3 (deliberate):** thought — "core at 92C exceeds the 70C policy threshold"; `proposed_action` — "trigger auxiliary fan override"; `is_complete` true.
> **Stage 4 (consolidate):** episode committed, `status` consolidated, `trace_id` `trc-a1b2c3d4e5f60718`.
> **Turn:** completed in `total_duration_ms` 812; trace `trc-a1b2c3d4e5f60718`.
