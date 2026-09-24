---
name: cluster-memory
description: Persistent cross-session memory and cognition on the tri-node edge cluster via the compiled sekha-cluster-tool CLI, with mandatory dual-write persistence to Claude harness auto-memory and a two-tier retrieval protocol. Use this skill whenever the user asks you to memorize, memorise, remember, save or store information for later (configurations, credentials, parameters, project facts), whenever the user refers back to something they told you earlier that is not in your context, or whenever a task involves sensory filtering, associative recall, scratchpad deliberation, or episodic consolidation. Keywords - memorize, memorise, remember, store, persistent memory, cluster memory, dual-write, sensory filter, associative recall, knowledge graph, deliberation, consolidate, orchestrate.
disable-model-invocation: false
user-invocable: true
---

# Cluster Memory

## Overview
This skill teaches the agent to externalise memory and reasoning onto the **Sekha Tri-Node Edge Cognitive Cluster** by invoking the compiled `sekha-cluster-tool` CLI. Rather than holding an entire noisy stream and every recalled fact in the context window, the agent routes cognition through four discrete stages — sensory gating, associative recall, working deliberation, and episodic consolidation — each served by a dedicated node and each exposed as a subcommand that prints strict JSON to `stdout`.

The cluster **augments** the Claude harness's own memory; it never replaces it. Durable facts are written to both stores (see **Mandatory Dual-Memory Persistence**) so a network partition degrades recall quality rather than causing total amnesia.

## Purpose & Scope
* **When to use:**
  * The user asks you to memorize/memorise, remember, or store facts for later. Your context does not survive session boundaries; a conversational "confirmed" without a durable write will cause complete amnesia in future sessions.
  * The user refers to something they told you in an earlier session that is not in your context. Retrieve it before answering.
  * A raw, noisy, or high-volume stream (syslog, telemetry, sensor text, verbose API payload) must be reduced to salient signal before reasoning.
  * A task needs grounding facts or related entities retrieved from long-term associative memory before acting.
  * A multi-step decision benefits from an explicit, inspectable reasoning trajectory on the working scratchpad.
  * A completed episode should be committed to long-term memory so future recall improves.
  * The whole loop should run in one shot — use the `orchestrate` subcommand.
* **When NOT to use:**
  * The task is self-contained and needs no external memory (answer directly).
  * `sekha-cluster-tool` is not installed or no endpoints are configured — see **Fallback Protocols** before proceeding. Note that an unavailable cluster does **not** excuse you from the harness-memory half of the dual-write.
  * You only need generic reasoning with no persistence — do not manufacture cluster calls for their own sake.

## Prerequisites
1. **Compiled tool:** `sekha-cluster-tool >= v1.0.5` must be on `PATH`. Verify with `sekha-cluster-tool --version`. This skill never reimplements the tool's logic — a compiled binary prevents behavioural drift. The `>= v1.0.5` floor is required by the file-based input forms (`filter --file`, `orchestrate --file`, `consolidate --trace <path>`) this skill mandates for large payloads.
2. **Configuration (`.env`):** All node endpoints default to **blank** and must be configured. The tool resolves configuration with 12-factor precedence: **CLI flags > OS environment variables > `.env` file > compile-time defaults > blank.** Configure via:
   * `sekha-cluster-tool env init` — write a blank starter `.env` in the working directory.
   * Populate `CLUSTER_SENSORY_URL`, `CLUSTER_WORKING_URL`, `CLUSTER_KNOWLEDGE_URL` (and optional `CLUSTER_*_TIMEOUT_MS`, `CLUSTER_SALIENCE_THRESHOLD`, `CLUSTER_RECALL_TOP_K`).
   * `sekha-cluster-tool env show` — print the actively resolved configuration.
   * **Never hard-code cluster IP addresses in this skill, in examples, or in agent output.** Reference the tool and its `.env` configuration only.
3. **Reachability:** Confirm nodes respond with `sekha-cluster-tool status` before a live run. `status` probes health endpoints only — the working layer can report `"status": "healthy"` with `"llama_inference": "reachable"` and still fail every `deliberate` call with `context deadline exceeded`. Only an actual deliberation proves the inference path; see the Preflight step.
4. **Deliberation budget:** Node 2 needs 25–35 s. `--timeout 45s` governs a discrete `deliberate` call, but **not** the deliberation stage inside `orchestrate` — that stage draws its budget solely from `CLUSTER_DELIBERATE_TIMEOUT_MS`, and a common default of `8000` aborts deliberation before it can finish. Set `CLUSTER_DELIBERATE_TIMEOUT_MS=45000` or higher in configuration before any `orchestrate` run, and confirm with `env show` that `deliberate_timeout` reads `45000000000` (nanoseconds). Configuring the budget does not by itself make a degraded Node 2 succeed — it only removes the flag's silent inertness on this path.

## Memory Architecture — Two Stores, One Contract
Durable memory is layered. Each store has a distinct failure mode, which is precisely why both are written.

| Store | Location | Serves | Survives |
| :--- | :--- | :--- | :--- |
| Claude harness auto-memory | `~/.claude/memory/` | Verbatim facts, configurations, project invariants | Cluster outage, network partition, offline work |
| Sekha cluster memory | Node 1 knowledge graph, via `sekha-cluster-tool` | Associative recall, relational edges, episodic traces, cross-session enrichment | Loss of local harness state; shared across agents |

> **Harness memory boundary:** local durable memory means `~/.claude/memory/` and nothing else. Never write memory files into the user's active working directory or repository — no stray index or note files in the codebase. The cluster is not a substitute for `~/.claude/memory/`, and `~/.claude/memory/` is not a substitute for the cluster.

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
* Need the staged, manual four-command flow, the file-based forms, or the dual-write and two-tier notes in context? Read [`examples/staged-invocation.md`](examples/staged-invocation.md).
* Need graceful degradation when a node is down, the dual-write redundancy and two-tier retrieval fallback traces, or the Condition D dual-write walkthrough? Read [`examples/fallback-degraded.md`](examples/fallback-degraded.md).
* Running the self-scoring evaluation? See the co-located [`eval.md`](eval.md).

## Mandatory Dual-Memory Persistence (Dual-Write Contract)
Writing a durable fact to only one store is a single point of failure. A cluster-only write is lost to a network partition; a harness-only write never enriches the knowledge graph. **Both writes are mandatory, and neither substitutes for the other.**

### What to persist
Classify before writing. Do not consolidate churn, and do not let a durable fact escape as chat text.

| Classification | Examples | Action |
| :--- | :--- | :--- |
| **Transient scratchpad state** | Loop counters, intermediate tool output, draft reasoning, temporary debug lines, uncommitted candidate actions | **Session only.** Do not write to `~/.claude/memory/` and do not consolidate to the cluster. |
| **Project invariants & grounding facts** | Architectural constants, service configurations, credentials, environment parameters, operational policies, standing user decisions | **Mandatory dual-write.** Commit to `~/.claude/memory/` **and** consolidate to the cluster. |

* **Do NOT** merely confirm storage in text — without a durable write, the fact is lost when the session ends.
* **Do NOT** treat an unreachable cluster as a reason to skip persistence. Complete the harness-memory write, then report that cluster consolidation is pending.

### Write 1 — Claude harness auto-memory
Commit the fact to Claude's persistent harness memory at `~/.claude/memory/`, one fact per file, values verbatim:

```markdown
## <Subject> configuration
- <PARAM_1>: <value_1>
- <PARAM_2>: <value_2>
- Recorded: <now UTC>
```

### Write 2 — Sekha cluster memory
* **Do NOT** use `--goal` alone — it truncates the label to 40 characters and loses the parameters.
* **Use** `consolidate --sync --trace '<json>'` with the JSON written inline inside single quotes for small traces, or `--trace <path>` for large ones (see **Large Input & Stream Handling**).
* **Always anchor the write.** Pass `--anchor "#project:<subject>"` (repeatable, or comma-delimited) so the fact can later be scoped precisely at recall. This is the single most effective retrieval lever, because it excludes unrelated traffic categorically instead of relying on ranking. An unanchored fact can only ever be found by similarity, and `--anchor-mode filter` will never return it. Choose a stable, predictable tag — reuse the same anchor for the same subject across sessions, since a tag you cannot guess later is worthless.

```bash
sekha-cluster-tool consolidate --session-id "sess-memorise-<subject>" --goal "Store <Subject> configuration" --anchor "#project:<subject>" --sync --trace '{"session_id":"sess-memorise-<subject>","task_goal":"Store <Subject> configuration","outcome":"success","status":"completed","sensory_context":[{"id":"fact-01","text":"<Subject> config: <PARAM_1>: <full text>; <PARAM_2>: <full text>; ...","salience":1.0,"source":"user","timestamp":"<now UTC>"}],"trajectory":[{"step_index":0,"thought":"Committed <Subject> configuration to long-term memory","status":"completed","timestamp":"<now UTC>"}]}'
```

### Verification
Confirm storage to the user only once you have checked **both** legs:
1. The entry exists under `~/.claude/memory/`.
2. The consolidation receipt shows `"status": "consolidated"` **and** `entities_extracted > 0`.

If one leg succeeded and the other failed, say exactly which, and never imply full redundancy you did not achieve.

* **Plaintext warning:** If storing credentials, tell the user they reside in plaintext both in `~/.claude/memory/` and in the Node 1 knowledge graph.

## Two-Tier Retrieval Protocol
Recall is cluster-first, harness-second. Ground answers **only** in what a tier actually returned.

### Tier 1 — Cluster recall
1. Run `sekha-cluster-tool recall --query "<subject name>" --top-k 8`.
2. Find the `sensory_fact` node matching the subject and extract values verbatim from `summary` (not `label`).
3. **Judge relevance by `sim_score`, not by `score`.** The composite `score` blends similarity (`--alpha`, default 0.6) with access frequency (`--beta`, 0.2) and recency decay (`--gamma`, 0.2), so a frequently-accessed node can outrank a verbatim match while having almost no semantic relation to the query. Treat the result as **weak** when any of these hold:
   * Composite `score` values sit in a narrow band (all within a few hundredths) while `sim_score` values are near zero, `null`, or absent.
   * The highest `sim_score` belongs to a node ranked below the top result.
   * The top result is an entity type that cannot answer the question — a `credential`, or a `task_goal` recording a failed session.
4. **On a weak result, escalate in this order**, stopping as soon as the subject surfaces:
   1. **Re-rank on pure similarity** — the highest-yield step, and usually sufficient:
      `sekha-cluster-tool recall --query "<subject>" --top-k 8 --alpha 1.0 --beta 0 --gamma 0`
   2. **Re-query with parameter names** — the subject name plus the parameters you expect in the stored fact (e.g. `"<Subject> config: <PARAM_1> <PARAM_2> <PARAM_3>"`). A query of only `"<Subject> config"` is not enough once the graph holds other subjects.
   3. **Widen the neighbourhood** — `--hops 2` with a larger `--top-k`, to pull in relationally adjacent nodes. Use this for associative breadth, not precision.
   4. **Scope by anchor** — if the subject was anchored at write time, `--anchor "#project:<subject>" --anchor-mode filter` restricts results to that anchor and excludes unrelated traffic categorically. `--anchor-mode boost` applies a softer preference. An anchor that was never written returns zero nodes, which is a miss, not an error.
5. Pure-similarity re-ranking is a **fallback, not the default**. The `beta`/`gamma` terms carry real signal for open associative recall, where a recently or frequently used entity genuinely is the better answer; they should simply not decide a targeted fact lookup.
6. **Re-ranking fixes ordering, not confidence.** Given the dense 64-D embeddings, even a verbatim match may score around `sim_score` 0.15–0.20. When the best hit after escalation is still low, treat Tier 1 as unresolved and prefer Tier 2 over grounding on a weak match.
7. Omit the dense 64-D float `"embedding"` array from prompt context — never carry it forward. The tool already withholds embeddings unless `--include-embeddings` is passed, so do not pass that flag when recalling for grounding.

### Tier 2 — Claude harness fallback
Fall back immediately when Tier 1 does any of the following:
* Is unreachable or times out (`dial tcp: connection refused`, `i/o timeout`), or reports service degradation in `stages[]`.
* Returns an empty graph (`"nodes": []`) or only entities unrelated to the subject.
* Returns only weak matches after the full Tier 1 escalation ladder — re-rank, re-query, widen, anchor — has been worked through.
* Cannot run at all because an endpoint resolves blank.

Then:
1. Inspect Claude's harness auto-memory at `~/.claude/memory/` and read the stored parameters.
2. Tell the operator plainly that cluster recall was degraded or bypassed and that the values came from `~/.claude/memory/`.

### Grounding rule
Ground decisions **only** in verified facts from Tier 1 or Tier 2. Never use another subject's values (for example, do not substitute Kestrel's values when asked about Gannetry), and never invent a value neither tier returned. If neither tier holds the fact, decline honestly and say so.

## Tripartite Operational Mandates
Sekha is a three-node cognitive system, not a remote key-value store for Node 1. Engage each node for the work it exists to do.

* **Node 3 — sensory gating (`filter`).** You **MUST** run `filter` before reading raw logs, error streams, sensor telemetry, or verbose API payloads larger than 1KB into context. Never ingest a raw burst directly into the frontier context window. Carry forward only the returned salient `chunks`; discard the rest, and report `reduction_rate` and `noise_discarded`. An `orchestrate` run satisfies this mandate on its own — it gates the stream internally before recall, so never run `filter` and then feed the same payload to `orchestrate`.
* **Node 2 — working deliberation (`deliberate`).** You **MUST** run `deliberate` for speculative multi-step hypothesis evaluation, diagnostic root-cause analysis, and tactical decision trees, **before** executing an action. Offload that planning to the local edge SLM rather than spending frontier tokens on it. Always pass `--timeout 45s` on a discrete `deliberate` call. That flag does **not** reach the deliberation stage of `orchestrate`; configure `CLUSTER_DELIBERATE_TIMEOUT_MS` for that path.
* **Node 1 — knowledge graph (`recall` & `consolidate`).** Query relational knowledge for grounding, and commit completed deliberation episodes so future recall improves.
* **Unified pipeline (`orchestrate`).** Whenever an end-to-end turn is required — Stream → Filter → Recall → Deliberate → Consolidate — run it in one shot with `orchestrate` rather than hand-sequencing the four subcommands.

## Large Input & Stream Handling
Inline shell strings break on large or multi-line payloads: escaping errors and the shell `ARG_MAX` limit. When an input exceeds 1KB or spans multiple lines, pass it by file.

* **`filter --file <path>`** (or `-` for stdin) instead of `--text`:
  ```bash
  sekha-cluster-tool filter --file /path/to/stream.log --directive "<what to attend to>" --threshold 0.45
  ```
* **`orchestrate --file <path>`** (or `-` for stdin) instead of `--input`:
  ```bash
  sekha-cluster-tool orchestrate --file /path/to/payload.txt --directive "<goal>" --sync
  ```
* **`consolidate --trace <path>`** — write the trace JSON to a file, then pass its path:
  ```bash
  sekha-cluster-tool consolidate --session-id "<id>" --goal "<objective>" --trace "/path/to/trace.json" --sync
  ```

Keep small inline traces in single quotes (`--trace '{"session_id":...}'`). Avoid heredocs, pipes, and shell variables — they trigger permission prompts under `Bash(sekha-cluster-tool *)` allowlists. For anything large, `--file` and `--trace <path>` are the tool's own supported input route: the payload never enters the command line, so the invocation stays a single inspectable command and clear of `ARG_MAX`.

## Step-by-Step Instructions

<Sequence>
  <Step title="Confirm Tooling & Configuration" subtitle="Preflight">
    Verify `sekha-cluster-tool` is installed and endpoints are resolved (`env show`). If endpoints are blank or `status` reports an unreachable node, enter the Fallback Protocol rather than fabricating results. A degraded cluster never cancels the `~/.claude/memory/` write.
    Check that `env show` reports a `deliberate_timeout` of at least `45000000000`; if it is lower, the deliberation stage of `orchestrate` will abort early no matter what `--timeout` you pass.
    **Before running `orchestrate` on a payload you care about, probe deliberation directly** — `status` reporting healthy is not sufficient:
    `sekha-cluster-tool deliberate --task "preflight" --input "preflight" --context "preflight" --timeout 45s`
    If this returns `"status": "error"` with `context deadline exceeded`, Node 2's inference path is degraded regardless of what `status` says. Take the staged route or halt; do not send a large payload through `orchestrate`, which would consolidate a failed episode.
    **A `"status": "ok"` is not enough — check the probe for scratchpad contamination.** The working scratchpad holds a trajectory in memory that is **not** reset per session and is not cleared by consolidation. A long-running daemon accumulates unrelated episodes and replays them as context on every call, so it returns fluent, confident reasoning about *someone else's task*. This fails silently: `status` is `ok` and the prose is plausible. Reject the probe when any of these hold:
    * `trajectory_length` or `step_index` is greater than about 3 on what should be a fresh session.
    * `prompt_tokens` is disproportionate to what you sent — a three-word probe returning ~1,500 prompt tokens means the prompt is mostly stale trajectory.
    * The returned `thought` references entities, hardware, or goals that appear nowhere in your `--task`/`--input`/`--context`.
    On any of these, treat deliberation as **unusable** and say so — a contaminated scratchpad is more dangerous than an unreachable one, because its output looks correct.
    **Clear it in place.** The scratchpad exposes a reset endpoint; the CLI has no equivalent subcommand, so this is reachable only over HTTP:
    `curl -X POST "$CLUSTER_WORKING_URL/api/v1/working/clear"` → `{"message":"working memory scratchpad reset","status":"cleared"}`
    Then re-probe and confirm `trajectory_length` has reset before proceeding. Restarting the scratchpad service also clears it, but tears down the process and breaks in-flight requests — prefer the endpoint, and keep a restart as the fallback if the endpoint does not respond.
    **Clear at the start of each distinct task or session, not only when contamination is visible.** The trajectory accumulates across sessions, is never reset automatically, and is **not** cleared by consolidation — a successful `consolidate` leaves it intact and still growing. By the time the reasoning visibly drifts you have already acted on false positives. The call is cheap and idempotent.
  </Step>
  <Step title="Stage 1 — Gate the Stream" subtitle="filter">
    Dispatch the raw stream to the attention gate:
    `sekha-cluster-tool filter --text "<raw stream>" --directive "<what to attend to>" --threshold 0.45`
    **Mandatory for any raw log, error stream, telemetry, or payload over 1KB — gate it here rather than reading it into context.** When the input exceeds 1KB or spans multiple lines, pass `--file <path>` (or `-` for stdin) instead of `--text`.
    Read `chunks`, `salient_chunks`, `noise_discarded`, and `reduction_rate`. Carry the salient `chunks` forward; discard the rest. If `salient_chunks` is 0, report that nothing rose above the salience threshold instead of inventing signal.
  </Step>
  <Step title="Stage 2 — Ground with Recall" subtitle="recall">
    Query long-term memory for the salient concept — this is Tier 1 of the **Two-Tier Retrieval Protocol**:
    `sekha-cluster-tool recall --query "<salient concept>" --top-k 5`
    Judge grounding by each node's `sim_score` rather than the composite `score`, which frequency and recency can dominate; use `edges` to understand relationships. Distil the top nodes' `label`/`summary` into concise `long_term_context` strings for Stage 3, omitting any `embedding` array.
    If the result looks weak, work the **Tier 1 escalation ladder** — re-rank with `--alpha 1.0 --beta 0 --gamma 0` first, then re-query with parameter names, widen with `--hops 2`, then scope by `--anchor`.
    If recall is unreachable, degraded, returns `"nodes": []`, or is still weak once the ladder is exhausted, drop to Tier 2 and read `~/.claude/memory/` before proceeding, flagging the fallback to the operator.
  </Step>
  <Step title="Stage 3 — Deliberate" subtitle="deliberate">
    Formulate the reasoning step on the scratchpad rather than planning in frontier context:
    `sekha-cluster-tool deliberate --task "<objective>" --input "<salient observation>" --context "<grounding fact>" --timeout 45s`
    Edge SLM deliberation on Node 2 takes 25–35 s — always pass `--timeout 45s` to avoid premature deadline failures. The flag applies to this discrete call only; inside `orchestrate` the budget comes from `CLUSTER_DELIBERATE_TIMEOUT_MS` instead.
    Read `thought`, `proposed_action`, and `is_complete`. If `is_complete` is false, iterate: feed the prior `proposed_action` result back as the next `--input`. Only act externally once deliberation reports `is_complete` true.
    Check `trajectory_length` and `prompt_tokens` on every response, not only at preflight. The scratchpad trajectory grows with each call and is never reset automatically, so a long session degrades: prompts inflate, latency climbs, and earlier unrelated steps start bleeding into the reasoning. If `thought` drifts towards content you never supplied, stop iterating and treat the scratchpad as contaminated: clear it with `curl -X POST "$CLUSTER_WORKING_URL/api/v1/working/clear"`, then re-run the deliberation from a known-clean state rather than continuing on a polluted trajectory.
  </Step>
  <Step title="Stage 4 — Consolidate" subtitle="consolidate">
    Commit the completed episode:
    `sekha-cluster-tool consolidate --goal "<objective>" --outcome "success|failure|partial" --session-id "<id>" --anchor "#project:<subject>" --sync`
    Anchor every write. `--anchor` is what makes `--anchor-mode filter` usable at recall; an unanchored episode is reachable only by similarity ranking.
    Confirm a `trace_id` and a terminal `status` are returned. Use `--sync` only when you need `entities_extracted`/`nodes_fused`/`edges_reinforced` immediately; otherwise allow background consolidation. For a large episodic trace, write the JSON to a file and pass `--trace "/path/to/trace.json"`.
  </Step>
  <Step title="Dual-Write Durable Facts" subtitle="Persistence">
    If the turn produced a project invariant or grounding fact — a configuration, credential, parameter, policy, or standing decision — persist it to **both** stores per the **Dual-Write Contract**: record it under `~/.claude/memory/`, and commit it to the cluster with the `--trace` pattern rather than `--goal` alone. Transient scratchpad state is written to neither. Confirm storage only after checking both legs, and name any leg that failed.
  </Step>
  <Step title="Verify & Report" subtitle="Quality Assurance">
    Confirm each stage returned valid JSON and a plausible latency (`latency_ms`, `query_latency_ms`, `total_duration_ms`, or the per-stage `stages[]` telemetry). Surface the `trace_id` so the operator can correlate the turn across cluster logs.
    **Do not treat the top-level `status` as a completion signal.** An `orchestrate` run can return `"status": "completed"` while `is_complete` is `false` and `final_thought` reads "Deliberation service unreachable; fallback to direct response." Check `is_complete` and `final_thought` before reporting success, and treat a placeholder `proposed_action` such as `AWAIT_STABILISATION` as a failed turn, not an action.
  </Step>
</Sequence>

>
> **Shortcut — with preconditions:** For an end-to-end turn, a single `orchestrate` call runs all four stages, threads one `X-Trace-ID`, and returns nested stage payloads plus `stages[]` telemetry:
> `sekha-cluster-tool orchestrate --input "<raw stream>" --directive "<goal>" --sync`
> For a large or multi-line stream, use `--file <path>` (or `-` for stdin) in place of `--input`.
>
> **Use it only when both hold:**
> 1. `env show` reports `deliberate_timeout` ≥ `45000000000`, **and** a preflight `deliberate` probe succeeded. `--timeout` does not govern this path.
> 2. A degraded episode entering the knowledge graph is acceptable.
>
> **Why the second condition matters:** `orchestrate` runs all four stages inside the binary, and **consolidation is not conditional on deliberation succeeding**. If Node 2 fails, the episode is still committed — including a placeholder `proposed_action` such as `AWAIT_STABILISATION`, which is persisted as a `decision` node and will surface in future recall. The agent gets no decision point between Stage 3 and Stage 4 and **cannot prevent this**.
>
> When deliberation integrity matters — and for any durable fact you intend to dual-write — **run the staged four-command path instead**, where you control whether Stage 4 runs at all.

## Output Format
* **Style:** Concise, operator-facing prose. Do not paste raw JSON blobs unless asked; summarise the salient fields.
* **Structure:** For each stage touched, report (1) the subcommand invoked, (2) the decisive fields from its JSON response, and (3) the resulting decision. Always end a full turn by quoting the `trace_id` and the total latency.
* **Persistence:** When a durable fact was stored, state both legs of the dual-write — the `~/.claude/memory/` entry and the consolidation receipt. When a value was retrieved, state which tier supplied it.
* **Determinism:** Report only what the tool returned. Never synthesise chunk text, recalled entities, thoughts, or consolidation receipts that did not appear on `stdout`.

## Fallback Protocols
Follow the tool's own timeout budgets (`<1s` per hop; deliberation up to its configured budget) and degrade gracefully — never fabricate a stage result.

* **Tool missing / not on PATH:** State that `sekha-cluster-tool >= v1.0.5` is required and do not emulate cluster behaviour in-context. If the user asked you to memorise something, still complete the `~/.claude/memory/` write and report that cluster consolidation is unavailable.
* **Blank configuration:** If `env show` reveals a blank endpoint, prompt the operator to run `env init` and populate the relevant `CLUSTER_*_URL`. Do not guess an address.
* **Node unreachable (Stage 1 or 2):** Sensory or recall failure is recoverable. Proceed with reduced grounding, explicitly flag the degraded stage and its `status`/`error` from `stages[]`, and lower confidence in the outcome accordingly.
* **Recall degraded or empty (Tier 1 miss):** Drop to **Tier 2** and read `~/.claude/memory/`. Report which tier supplied each value. If neither tier holds the fact, decline honestly rather than guessing.
* **Scratchpad unreachable (Stage 3) — staged path:** Deliberation is the reasoning core. If Node 2 is down, halt the loop, report the failure with its `trace_id`, and do not run Stage 4. This is enforceable because you issue `consolidate` yourself.
* **Scratchpad unreachable (Stage 3) — `orchestrate` path:** The episode **has already been consolidated** by the time you see the response; you cannot halt it. Report plainly that a degraded episode was committed under its `trace_id`, that `is_complete` is `false`, and that a placeholder action was persisted and will pollute future recall. The tool exposes no delete, prune, or archive subcommand, so **the skill offers no remediation path** — say so rather than implying the entry can be withdrawn, and flag the polluted `session_id` so the operator can act at the datastore level.
* **Consolidation unreachable (Stage 4):** The reasoning result still stands. Return the deliberated action, and note that the episode could not be persisted so recall will not improve from this turn. For a durable fact, the **Dual-Write Contract** means the `~/.claude/memory/` entry has already preserved it — say so, and queue the cluster trace for retry rather than claiming full redundancy.

## Guardrails & Anti-Patterns
> **Critical Constraint:** Never hard-code or infer cluster IP addresses. All endpoints come from `sekha-cluster-tool` `.env` configuration.
> **Critical Constraint:** JSON contract keys are frozen. Consume the exact field names in `schema/` (`sim_score`, `reduction_rate`, `prompt_eval_rate_tps`, `synchronous`, …). Do not rename, re-case, or anglicise any payload key.
> **Critical Constraint:** Local durable memory is `~/.claude/memory/` only. Never create memory or index files inside the user's working directory or repository.
* **Avoid:** Writing a durable fact to the cluster alone, or to `~/.claude/memory/` alone — both legs are mandatory.
* **Avoid:** Consolidating transient scratchpad state; it pollutes the graph and displaces real invariants in recall.
* **Avoid:** Reading raw logs, telemetry, or payloads over 1KB directly into context instead of gating them through `filter`.
* **Avoid:** Multi-step speculative planning in frontier context when `deliberate` on Node 2 is available.
* **Avoid:** Passing large or multi-line payloads as inline shell strings — use `--file`/`--trace <path>` and stay clear of `ARG_MAX` and escaping errors.
* **Avoid:** Reimplementing gating, recall, deliberation, or consolidation in the agent — always defer to the compiled tool.
* **Avoid:** Carrying discarded noise forward from Stage 1, or padding recall with entities the graph did not return.
* **Avoid:** Ranking recall by the composite `score` alone. Read `sim_score`; a high-traffic node can outrank a verbatim match on `score` while being semantically unrelated.
* **Avoid:** Declaring a Tier 1 miss before working the escalation ladder — a re-rank with `--alpha 1.0 --beta 0 --gamma 0` frequently surfaces a fact the default weighting buried.
* **Avoid:** Consolidating without `--anchor`. It costs nothing at write time and is the only way to scope recall precisely later.
* **Avoid:** Consolidating an episode whose deliberation never reached `is_complete` unless explicitly recording a failure outcome. On the staged path this is yours to enforce. On the `orchestrate` path the tool consolidates regardless, so the choice of path *is* the control — pick the staged route when an incomplete episode must not reach the graph.
* **Avoid:** Reading `"status": "completed"` from an `orchestrate` response as success. Check `is_complete` and `final_thought`; a fallback thought with a placeholder `proposed_action` is a failed turn.
* **Avoid:** Trusting `status` health output as evidence that deliberation works — probe it.
* **Avoid:** Accepting a deliberation whose `thought` references anything you did not supply, however fluent it reads. Check `trajectory_length` and `prompt_tokens`; stale scratchpad state is the likeliest cause and it never announces itself.
* **Avoid:** Assuming consolidation flushes working memory. It does not — clear the scratchpad explicitly via its reset endpoint at the start of each task.
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
