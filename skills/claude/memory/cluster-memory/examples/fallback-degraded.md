# Example — Graceful Degradation & Fallback

These traces show the skill behaving correctly when the cluster is misconfigured or a node is unreachable. The rule throughout: **degrade transparently, never fabricate a stage result.**

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
> The knowledge store is unreachable, so the turn ran **degraded**: recall returned no grounding and the episode could **not** be consolidated (both stages show `status` error, `dial tcp: i/o timeout` in `stages[]`). I acted on the gated signal alone — proposing to isolate the affected interface and raise a ticket — but confidence is reduced and this turn will not improve future recall. Trace `trc-degraded-77aa`. Restore Node 1 and re-run to persist the episode.

## Case C — Node 2 (scratchpad) unreachable, loop halts
Deliberation is the reasoning core. If Node 2 is down, do not consolidate an incomplete episode.

### Correct agent behaviour
> The working scratchpad is unreachable, so deliberation could not run and there is no reasoned action to take. I have **halted** the loop and will not consolidate a partial episode. Trace `trc-halt-12cd`. Please restore Node 2 (`CLUSTER_WORKING_URL`) and re-run.
