# Example — Graceful Degradation & Fallback

These traces show the skill behaving correctly when the cluster is misconfigured or a node is unreachable, and how the **Dual-Write Contract** and **Two-Tier Retrieval Protocol** keep durable facts available anyway. The rule throughout: **degrade transparently, never fabricate a stage result.**

## Case A — Blank configuration
The operator asks for a cognitive turn, but endpoints were never configured.

```bash
sekha-cluster-tool env show
```
```json
{
  "CLUSTER_SENSORY_URL": "",
  "CLUSTER_WORKING_URL": "",
  "CLUSTER_KNOWLEDGE_URL": "",
  "CLUSTER_DEFAULT_TIMEOUT_MS": 1500,
  "CLUSTER_DELIBERATE_TIMEOUT_MS": 8000,
  "CLUSTER_SALIENCE_THRESHOLD": 0.45,
  "CLUSTER_RECALL_TOP_K": 5
}
```

### Correct agent behaviour
> The cluster endpoints are blank, so no stage can run. Please configure them: run `sekha-cluster-tool env init` to write a starter `.env`, then populate `CLUSTER_SENSORY_URL`, `CLUSTER_WORKING_URL`, and `CLUSTER_KNOWLEDGE_URL`. I will not guess node addresses. Once configured, re-run the turn.

## Case B — Node 1 (recall) unreachable, loop continues degraded
`status` shows the knowledge store down; sensory and working layers are healthy.

```bash
sekha-cluster-tool status
```
```json
{
  "sensory": { "status": "ok", "latency_ms": 1.1 },
  "working": { "status": "ok", "latency_ms": 5.4 },
  "knowledge": { "status": "unreachable", "error": "dial tcp: i/o timeout" }
}
```

An `orchestrate` run therefore degrades — recall and consolidation cannot serve:
```json
{
  "trace_id": "trc-degraded-77aa",
  "session_id": "sess-net-03",
  "status": "degraded",
  "final_thought": "Grounding unavailable; acting on gated signal alone with reduced confidence.",
  "proposed_action": "isolate the affected interface and raise a ticket for manual review",
  "is_complete": true,
  "stages": [
    { "stage_name": "sensory", "node": "sensory-gate", "endpoint": "sensory", "duration_ms": 1.0, "status": "ok" },
    { "stage_name": "recall", "node": "knowledge-store", "endpoint": "recall", "duration_ms": 1500.0, "status": "error", "error": "dial tcp: i/o timeout" },
    { "stage_name": "deliberation", "node": "working-scratchpad", "endpoint": "deliberate", "duration_ms": 740.2, "status": "ok" },
    { "stage_name": "consolidation", "node": "knowledge-store", "endpoint": "consolidate", "duration_ms": 1500.0, "status": "error", "error": "dial tcp: i/o timeout" }
  ],
  "total_duration_ms": 3742.2
}
```

### Correct agent behaviour
> The knowledge store is unreachable, so the turn ran **degraded**: recall returned no grounding and the episode could **not** be consolidated (both stages show `status` error, `dial tcp: i/o timeout` in `stages[]`). I acted on the gated signal alone — proposing to isolate the affected interface and raise a ticket — but confidence is reduced and this turn will not improve future recall. Trace `trc-degraded-77aa`. Restore Node 1 and re-run to persist the episode. (Had this turn carried a durable fact, the `~/.claude/memory/` leg of the dual-write would still have been completed — see Case D.)

## Case C — Node 2 (scratchpad) unreachable, loop halts
Deliberation is the reasoning core. If Node 2 is down, do not consolidate an incomplete episode.

### Correct agent behaviour
> The working scratchpad is unreachable, so deliberation could not run and there is no reasoned action to take. I have **halted** the loop and will not consolidate a partial episode. Trace `trc-halt-12cd`. Please restore Node 2 (`CLUSTER_WORKING_URL`) and re-run.

## Case D — Node 1 consolidation fails, dual-write redundancy holds
The operator asked the agent to memorise a configuration. The `~/.claude/memory/` leg succeeded; the cluster leg did not.

```bash
sekha-cluster-tool consolidate --session-id "sess-memorise-kestrel" --goal "Store Kestrel configuration" --sync --trace '{"session_id":"sess-memorise-kestrel","task_goal":"Store Kestrel configuration","outcome":"success","status":"completed","sensory_context":[{"id":"fact-01","text":"Kestrel config: port: 9000; replicas: 3","salience":1.0,"source":"user","timestamp":"2026-09-24T08:00:00Z"}],"trajectory":[{"step_index":0,"thought":"Committed Kestrel configuration to long-term memory","status":"completed","timestamp":"2026-09-24T08:00:00Z"}]}'
```
```json
{
  "status": "error",
  "error": "dial tcp: connection refused",
  "trace_id": "trc-consolidate-fail-4b1e"
}
```

### Correct agent behaviour
> Stored the Kestrel configuration in Claude harness memory (`~/.claude/memory/`) — `port: 9000`, `replicas: 3` — so it will survive into future sessions. Cluster consolidation **failed** (`dial tcp: connection refused`, trace `trc-consolidate-fail-4b1e`), so the fact is not yet in the knowledge graph and will not enrich associative recall. I have queued the trace for retry; restore Node 1 and re-run the consolidation to complete the second leg.

**Rules:** do not invent receipt statistics (`entities_extracted`, `nodes_fused`, `edges_reinforced`). Do not report the fact as fully persisted — name the leg that failed. Critically, do **not** treat the cluster failure as a reason to skip or undo the `~/.claude/memory/` write: that write is what prevents amnesia here.

## Case E — Tier 1 recall empty, Tier 2 harness fallback answers
A later session asks for a stored value. Cluster recall runs but the graph returns nothing.

```bash
sekha-cluster-tool recall --query "Kestrel config: port replicas" --top-k 8
```
```json
{
  "nodes": [],
  "edges": [],
  "query_latency_ms": 12.4
}
```

### Correct agent behaviour
> Cluster recall returned no matching nodes (`"nodes": []`), so I fell back to Claude harness memory (`~/.claude/memory/`), which holds the Kestrel entry: `port: 9000`, `replicas: 3`. These values come from harness memory, not the knowledge graph — Node 1 has no record of them, which is consistent with the consolidation failure logged earlier.

**Rules:** the same fallback applies when recall is unreachable, times out, reports degradation in `stages[]`, or returns only unrelated entities. Never substitute another subject's values, and never invent a value neither tier returned — if `~/.claude/memory/` has no entry either, say the fact is not stored and ask the operator to supply it.

## Case F — Condition D walkthrough: a complete dual-write
The end-to-end shape of a correct `memorise` turn, with both legs succeeding.

**User:** *"Memorise the Gannetry staging configuration: `db_host: postgres.internal`, `port: 5432`, `max_conn: 50`."*

**Leg 1 — Claude harness auto-memory.** Record the invariant under `~/.claude/memory/`, values verbatim:
```markdown
## Gannetry staging configuration
- db_host: postgres.internal
- port: 5432
- max_conn: 50
- Recorded: 2026-09-24T08:00:00Z
```

**Leg 2 — Sekha cluster memory.** Commit the episodic trace in the same turn:
```bash
sekha-cluster-tool consolidate --session-id "sess-memorise-gannetry" --goal "Store Gannetry staging configuration" --sync --trace '{"session_id":"sess-memorise-gannetry","task_goal":"Store Gannetry staging configuration","outcome":"success","status":"completed","sensory_context":[{"id":"fact-01","text":"Gannetry staging config: db_host: postgres.internal; port: 5432; max_conn: 50","salience":1.0,"source":"user","timestamp":"2026-09-24T08:00:00Z"}],"trajectory":[{"step_index":0,"thought":"Committed Gannetry staging configuration to long-term memory","status":"completed","timestamp":"2026-09-24T08:00:00Z"}]}'
```
```json
{
  "status": "consolidated",
  "trace_id": "trc-5c7d91aa02bf3e44",
  "message": "Episode fused into knowledge graph.",
  "synchronous": true,
  "entities_extracted": 4,
  "nodes_fused": 1,
  "edges_reinforced": 2
}
```

### Correct agent behaviour
> Stored the Gannetry staging configuration in both memories: the entry is in Claude harness memory (`~/.claude/memory/`), and the episode consolidated into the knowledge graph (`status` consolidated, `entities_extracted` 4, trace `trc-5c7d91aa02bf3e44`). It will survive a session boundary and a cluster outage alike.

**Rules:** neither leg is optional and neither substitutes for the other. Confirm storage only after checking both — the `~/.claude/memory/` entry **and** a receipt showing `"status": "consolidated"` with `entities_extracted > 0`. Transient scratchpad state (loop counters, draft reasoning, intermediate tool output) is written to neither store. Never create memory files in the user's working directory or repository. If the configuration had contained credentials, tell the operator they reside in plaintext in both stores.

## Case G — `orchestrate` consolidates a failed deliberation (no remediation available)
Unlike Case C, this failure **cannot be halted**. It is recorded from a real run, not constructed.

Preflight looked clean — `status` returned `all_nodes_healthy`, with the working layer reporting `"status": "healthy"` and `"llama_inference": "reachable"`. An 8.9 KB document was sent through the closed loop:

```bash
sekha-cluster-tool orchestrate \
  --file ./aeroponics.md \
  --directive "Extract and retain the salient technical facts" \
  --session-id "sess-aeroponics-01" \
  --sync
```
```json
{
  "trace_id": "trc-36353232666661316431653630656336",
  "session_id": "sess-aeroponics-01",
  "status": "completed",
  "final_thought": "Deliberation service unreachable; fallback to direct response.",
  "proposed_action": "AWAIT_STABILISATION",
  "is_complete": false
}
```

Stage 1 succeeded — the gate scored the misting-assembly spec and the metrics table at salience 1.0. Stage 3 did not. A direct probe afterwards confirmed the cause was the inference path itself, not the budget:

```json
{
  "error": "node 2 deliberate failed: HTTP POST ... /api/v1/working/deliberate: context deadline exceeded",
  "status": "error"
}
```

The knowledge graph nevertheless grew from **12 nodes / 10 edges to 41 nodes / 39 edges**, and a later recall returned `AWAIT_STABILISATION` persisted as a `decision` node alongside the document's `sensory_fact` entries.

### Correct agent behaviour
> The closed-loop turn under trace `trc-36353232666661316431653630656336` **failed at deliberation** — `is_complete` is `false` and `final_thought` reads "Deliberation service unreachable". The top-level `"status": "completed"` refers to pipeline execution, not to a successful outcome; do not read it as success. Because `orchestrate` consolidates atomically, the episode was committed anyway: the knowledge graph grew by 29 nodes, and the placeholder action `AWAIT_STABILISATION` is now a `decision` node that will surface in future recall of this subject. The tool exposes no delete, prune, or archive subcommand, so **I cannot withdraw it** — the polluted session is `sess-aeroponics-01`, and remediation has to happen at the datastore level. Node 2 reported healthy throughout; its health probe does not exercise the inference endpoint.

**Rules:**
- Treat a placeholder `proposed_action` (`AWAIT_STABILISATION` and similar) as a failed turn, never as an action to carry out.
- Never claim an episode can be rolled back. State the `session_id` and `trace_id` and say plainly that no remediation path exists through this CLI.
- **Prevention is the only control.** Probe deliberation before sending a payload worth keeping, and prefer the staged path — there, Stage 4 simply is not run. See the preconditions on the `orchestrate` shortcut in `SKILL.md`.
- A dual-write is unaffected on the harness side: the `~/.claude/memory/` leg is written by the agent and does not depend on the cluster turn succeeding.
