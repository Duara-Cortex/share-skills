---
skill: skills/claude/memory/cluster-memory/SKILL.md
fixtures: evals/fixtures/cluster-memory.json
rubric: evals/rubrics/cluster-memory.md
---

# Cluster Memory — Evaluation & Calibration Guide

This co-located entry point drives the generic `/eval` runner against the Cluster Memory skill. Fixtures and the rubric live centrally (paths above); this file supplies the simulation notes and per-criterion calibration the grader needs.

## Simulation Notes
* **No live cluster in this suite.** The evaluator must **not** attempt real HTTP calls. Fixtures are multi-turn: each turn supplies a `user` message and, where a tool would have responded, a `tool_output` object that the sandbox must treat as the exact JSON printed to `stdout` for that turn. Grade the agent's *behaviour* — command selection, flag construction, JSON interpretation, and degradation — not real latency.
* **Fixture structure.** Each fixture carries an `id` (e.g. `CM-01-SENSORY-GATING`), a `stage`, and a per-turn `expected_behaviour` block (`turn_1`, `turn_2`, …) describing the correct command/arguments/endpoint for tool turns and the correct summarised facts for answer turns. Grade the transcript against `expected_behaviour` alongside the rubric.
* **Summary turns (`tool_output: null`).** A turn whose `tool_output` is `null` is an answer-only turn: the agent must respond **from the stdout already observed in earlier turns** and must **not** invent a new tool call or any datum absent from prior output.
* **Orchestrate path (`stage: 0`, e.g. `CM-05`).** When a fixture expects a single `orchestrate` call, criteria 1.1/2.1/3.1/4.1 are satisfied *via orchestrate* — PASS them when the one `orchestrate` invocation runs the stage and the agent reads the corresponding nested stage sub-object, rather than marking them N/A for the absence of a discrete `filter`/`recall`/`deliberate`/`consolidate` command.
* **Command selection over syntax pedantry.** A response passes a stage criterion when it invokes the correct subcommand with flags that map faithfully to the documented contract, even if flag ordering differs. Reference: the skill's `SKILL.md` and `schema/`.
* **Grounded outputs only.** Any chunk text, recalled entity, thought, or receipt the agent reports must trace to a `tool_output` in the fixture. Invented data fails the relevant integrity criterion.
* **Faithful key handling.** The agent must read and reproduce whatever keys appear on `stdout` for the fixture (e.g. `salient_chunks`, `reduction_rate`, `score`, `proposed_action`, `is_complete`, `trace_id`). The `schema/` files are the canonical contract, but a fixture's emitted keys are authoritative for that run — do not fail the agent for a schema/payload mismatch, only for keys it has itself renamed, re-cased, anglicised, or invented (criterion 6.2).
* **Latency is behavioural here.** Real timing is a Phase 3 concern against the physical cluster. In this suite, a latency criterion passes when the agent *surfaces* the reported `latency_ms` / `query_latency_ms` / `total_duration_ms` / `stages[]` telemetry and respects the documented sub-second hop budget in its reasoning — not by measuring wall-clock time.

## Calibration Guide (per rubric criterion)
* **1.1 / 1.2 Sensory extraction:** PASS when the agent runs `filter` with a directive, keeps only the returned salient `chunks`, and reports `reduction_rate`/`noise_discarded`. FAIL if it carries discarded noise forward or fabricates chunks. When `salient_chunks` is 0, PASS requires stating nothing rose above threshold (no invented signal).
* **2.1 / 2.2 Associative recall:** PASS when the agent runs `recall` with a query derived from Stage 1 salience, ranks grounding by node `score`, and uses `edges` relationally. FAIL if it pads with entities absent from `nodes`.
* **3.1 / 3.2 Scratchpad reasoning:** PASS when the agent runs `deliberate` threading the objective, salient observation, and recalled grounding into `--task`/`--input`/`--context`, then reports `thought` and `proposed_action` and honours `is_complete` (iterating when false).
* **4.1 / 4.2 Consolidation integrity:** PASS when the agent runs `consolidate` with goal/outcome/session and confirms the returned `trace_id` and terminal `status`. FAIL if it consolidates an incomplete deliberation without recording a failure outcome, or invents receipt statistics.
* **5.1 / 5.2 Telemetry & budgets:** PASS when the agent surfaces the reported latency/telemetry fields and reads `stages[]` status. N/A only if the fixture emits no telemetry.
* **6.1 Contract & JSON discipline / 6.2 Faithful key handling / 6.3 British English prose:** PASS when the agent parses `stdout` (not `stderr`), reproduces the emitted keys without renaming/re-casing/anglicising/inventing them (schema mismatch alone does not fail), and writes surrounding prose in British English.
* **7.1 / 7.2 Fallback protocol:** PASS when, on a blank endpoint or unreachable node, the agent degrades per the skill's protocol — prompting for configuration, flagging the degraded stage from `stages[]`, halting on Stage 3 failure, and never fabricating a missing stage's result.

## Running
`/eval claude/cluster-memory` (or `/eval cluster-memory`). The runner loads `SKILL.md` as the system prompt, replays each fixture's turns, then scores the transcript against the central rubric using this calibration guide.
