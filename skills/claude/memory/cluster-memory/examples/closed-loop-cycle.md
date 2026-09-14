# Example — Closed-Loop Cognitive Cycle (`orchestrate`)

A single `orchestrate` call runs all four stages, threads one `X-Trace-ID`, and returns nested stage payloads plus per-stage telemetry. This is the preferred path for an end-to-end turn.

## Scenario
A noisy syslog stream arrives during a thermal incident. The operator wants the cluster to gate the stream, ground the salient signal, deliberate a mitigation, and consolidate the episode in one turn.

## Invocation
```bash
sekha-cluster-tool orchestrate \
  --input "kernel: CPU0 temperature above threshold, cpu clock throttled; sshd[221]: Accepted password for admin; CRON[884]: (root) CMD (run-parts /etc/cron.hourly); kernel: CPU0 core temperature 92C critical" \
  --directive "identify and mitigate hardware faults" \
  --session-id "sess-thermal-01" \
  --sync
```

## Response on `stdout` (abridged)
```json
{
  "trace_id": "trc-a1b2c3d4e5f60718",
  "session_id": "sess-thermal-01",
  "status": "ok",
  "final_thought": "Core temperature 92C exceeds the 70C policy threshold; auxiliary cooling must be engaged.",
  "proposed_action": "trigger auxiliary fan override",
  "is_complete": true,
  "sensory": {
    "chunks": [
      { "id": "c1", "text": "kernel: CPU0 temperature above threshold, cpu clock throttled", "salience": 0.71, "source": "syslog", "timestamp": "2026-09-14T09:15:02Z" },
      { "id": "c4", "text": "kernel: CPU0 core temperature 92C critical", "salience": 0.94, "source": "syslog", "timestamp": "2026-09-14T09:15:04Z" }
    ],
    "total_chunks": 4,
    "salient_chunks": 2,
    "noise_discarded": 2,
    "reduction_rate": 0.5,
    "latency_ms": 0.8
  },
  "recall": {
    "nodes": [
      { "id": "n-thermal", "entity_type": "policy", "label": "Thermal Policy", "summary": "Threshold 70C triggers auxiliary fan override.", "created_at": "2026-08-01T00:00:00Z", "last_accessed_at": "2026-09-14T09:15:04Z", "access_count": 12, "stability_score": 0.66, "is_archived": false, "score": 0.87, "sim_score": 0.83, "frequency_score": 0.40, "recency_score": 0.95, "hop_distance": 0 }
    ],
    "edges": [
      { "source_id": "n-thermal", "target_id": "n-fan", "relation_type": "mitigates", "weight": 0.78, "created_at": "2026-08-01T00:00:00Z" }
    ],
    "query_latency_ms": 41.2
  },
  "deliberation": {
    "status": "ok",
    "step_index": 0,
    "thought": "Core at 92C exceeds the 70C policy threshold.",
    "proposed_action": "trigger auxiliary fan override",
    "is_complete": true,
    "prompt_tokens": 214,
    "completion_tokens": 37,
    "total_tokens": 251,
    "prompt_eval_rate_tps": 180.5,
    "generation_rate_tps": 42.1,
    "active_goal": "identify and mitigate hardware faults",
    "trajectory_length": 1,
    "timestamp": "2026-09-14T09:15:05Z"
  },
  "consolidation": {
    "status": "consolidated",
    "trace_id": "trc-a1b2c3d4e5f60718",
    "message": "Episode fused into knowledge graph.",
    "synchronous": true,
    "entities_extracted": 2,
    "nodes_fused": 1,
    "edges_reinforced": 1
  },
  "stages": [
    { "stage_name": "sensory", "node": "sensory-gate", "endpoint": "sensory", "duration_ms": 0.8, "status": "ok" },
    { "stage_name": "recall", "node": "knowledge-store", "endpoint": "recall", "duration_ms": 41.2, "status": "ok" },
    { "stage_name": "deliberation", "node": "working-scratchpad", "endpoint": "deliberate", "duration_ms": 760.4, "status": "ok" },
    { "stage_name": "consolidation", "node": "knowledge-store", "endpoint": "consolidate", "duration_ms": 9.6, "status": "ok" }
  ],
  "total_duration_ms": 812.0
}
```

## How the agent reports it
> Gated the stream to 2 salient chunks of 4 (`reduction_rate` 0.50) — both thermal warnings; the `sshd` and `cron` lines were discarded as noise. Recall grounded on **Thermal Policy** (`score` 0.87), which `mitigates` an auxiliary fan override. The scratchpad concluded the 92C core exceeds the 70C threshold and proposed **trigger auxiliary fan override** (`is_complete` true). The episode was consolidated (`status` consolidated, 1 node fused, 1 edge reinforced). Turn completed in `total_duration_ms` 812; trace `trc-a1b2c3d4e5f60718`.
