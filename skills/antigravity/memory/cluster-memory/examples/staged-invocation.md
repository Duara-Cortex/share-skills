# Example: Staged Cognitive Invocation

This walkthrough illustrates how to execute discrete, step-by-step cognitive invocations across individual cluster nodes when fine-grained control or intermediate intervention is required.

---

## Stage 1: Sensory Filtering on Node 3

When high-frequency sensory observations or syslog bursts arrive, filter the stream using Node 3 to eliminate background noise:

```bash
sekha-cluster-tool filter \
  --text "[2026-09-14T20:10:01Z] node-1 heartbeat ok [2026-09-14T20:10:04Z] ALERT: pwr-rail-4 voltage dropped below 11.2V [2026-09-14T20:10:06Z] routine fan check ok" \
  --directive "Identify power infrastructure anomalies" \
  --threshold 0.45
```

### Response (`stdout`)
```json
{
  "chunks": [
    {
      "id": "chnk-001",
      "text": "ALERT: pwr-rail-4 voltage dropped below 11.2V",
      "salience": 0.94,
      "source": "telemetry-bus",
      "timestamp": "2026-09-14T20:10:04Z"
    }
  ],
  "total_chunks": 3,
  "salient_chunks": 1,
  "noise_discarded": 2,
  "reduction_rate": 0.667,
  "latency_ms": 0.82
}
```

---

## Stage 2: Associative Recall Grounding on Node 1

Extract the salient text (`ALERT: pwr-rail-4 voltage dropped below 11.2V`) and ground the concept in the long-term knowledge graph:

```bash
sekha-cluster-tool recall \
  --query "pwr-rail-4 undervoltage failure runbook" \
  --top-k 2
```

### Response (`stdout`)
```json
{
  "nodes": [
    {
      "id": "ent-pwr-policy-04",
      "entity_type": "policy",
      "label": "Power Rail Redundancy Policy",
      "summary": "Mandates immediate auxiliary bus transfer if rail drops below 11.4V for more than 500ms.",
      "created_at": "2026-01-15T08:00:00Z",
      "last_accessed_at": "2026-09-12T14:22:10Z",
      "access_count": 42,
      "stability_score": 0.88,
      "is_archived": false,
      "score": 0.912,
      "sim_score": 0.89,
      "frequency_score": 0.95,
      "recency_score": 0.92,
      "hop_distance": 0
    },
    {
      "id": "ent-hw-aux-bus",
      "entity_type": "device",
      "label": "Auxiliary Power Bus B",
      "summary": "Secondary standby DC supply for edge compute nodes.",
      "created_at": "2026-01-15T08:00:00Z",
      "last_accessed_at": "2026-09-10T11:05:00Z",
      "access_count": 18,
      "stability_score": 0.76,
      "is_archived": false,
      "score": 0.785,
      "sim_score": 0.74,
      "frequency_score": 0.82,
      "recency_score": 0.85,
      "hop_distance": 1
    }
  ],
  "edges": [
    {
      "source_id": "ent-pwr-policy-04",
      "target_id": "ent-hw-aux-bus",
      "relation_type": "activates",
      "weight": 0.95,
      "created_at": "2026-01-15T08:00:00Z",
      "last_reinforced_at": "2026-09-12T14:22:10Z"
    }
  ],
  "query_latency_ms": 1.45
}
```

---

## Stage 3: Working Memory Deliberation on Node 2

Format the task objective, salient observation, and distilled graph facts into the scratchpad:

```bash
sekha-cluster-tool deliberate \
  --task "Engage auxiliary power bus" \
  --input "pwr-rail-4 measured at 11.18V for 750ms" \
  --context "[policy: Power Rail Redundancy Policy] Transfer to Auxiliary Power Bus B"
```

### Response (`stdout`)
```json
{
  "status": "ok",
  "step_index": 0,
  "thought": "Voltage on pwr-rail-4 has breached 11.4V threshold for >500ms (11.18V for 750ms). In accordance with the Power Rail Redundancy Policy, the primary supply is unstable. Action required: command transfer switch to Auxiliary Power Bus B immediately.",
  "proposed_action": "switch_auxiliary_bus(target='Auxiliary Power Bus B', isolate_primary='pwr-rail-4')",
  "is_complete": true,
  "candidate_actions": [
    {
      "id": "act-01",
      "type": "hardware_command",
      "payload": {
        "action": "transfer_power",
        "target": "auxiliary_bus_b"
      },
      "committed": false,
      "created_at": "2026-09-14T20:12:00Z"
    }
  ],
  "prompt_tokens": 148,
  "completion_tokens": 62,
  "total_tokens": 210,
  "prompt_eval_rate_tps": 420.5,
  "generation_rate_tps": 48.2,
  "active_goal": "Engage auxiliary power bus",
  "trajectory_length": 1,
  "timestamp": "2026-09-14T20:12:00Z"
}
```

---

## Stage 4: Episodic Consolidation on Node 1

Once the action has been committed, persist the session outcome back to the long-term knowledge graph for Hebbian reinforcement:

```bash
sekha-cluster-tool consolidate \
  --goal "Engage auxiliary power bus" \
  --outcome "success" \
  --session-id "sess-pwr-20260914" \
  --sync
```

### Response (`stdout`)
```json
{
  "status": "consolidated",
  "trace_id": "trc-pwr-20260914",
  "episode_id": "ep-20260914-8831",
  "session_id": "sess-pwr-20260914",
  "nodes_updated": 2,
  "edges_reinforced": 1,
  "entities_extracted": 0,
  "nodes_fused": 0,
  "decay_applied": true,
  "latency_ms": 3.12
}
```
