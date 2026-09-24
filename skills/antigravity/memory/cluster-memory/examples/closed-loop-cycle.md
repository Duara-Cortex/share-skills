# Example: Closed-Loop Cognitive Cycle

This walkthrough demonstrates an end-to-end cognitive response executed through the unified `sekha-cluster-tool orchestrate` subcommand.

---

## 1. Context & Scenario

An edge sensory collector receives an anomalous hardware event on the telemetry bus. The agent is tasked with filtering the stream, grounding the alert in long-term memory, deliberating a mitigation action via the local working scratchpad, and consolidating the episode into the knowledge graph.

---

## 2. Execution Command

```bash
sekha-cluster-tool orchestrate \
  --input "CRITICAL kernel panic risk: nvme0n1 write latency spiked to 4200ms, queue depth 64" \
  --directive "Resolve storage latency crisis" \
  --anchor "#infrastructure:storage" \
  --trace-id "trc-stor-0099" \
  --sync
```

---

## 3. Emitted Output Contract (`stdout`)

```json
{
  "trace_id": "trc-stor-0099",
  "status": "success",
  "stages": [
    {
      "stage_index": 1,
      "name": "filter",
      "status": "ok",
      "node": "Node 3 (Sensory Filter)",
      "duration_ms": 0.95
    },
    {
      "stage_index": 2,
      "name": "recall",
      "status": "ok",
      "node": "Node 1 (Knowledge Store)",
      "duration_ms": 1.62
    },
    {
      "stage_index": 3,
      "name": "deliberate",
      "status": "ok",
      "node": "Node 2 (Working Scratchpad)",
      "duration_ms": 178.50
    },
    {
      "stage_index": 4,
      "name": "consolidate",
      "status": "ok",
      "node": "Node 1 (Episodic Store)",
      "duration_ms": 2.85
    }
  ],
  "filter": {
    "chunks": [
      {
        "id": "chnk-stor-01",
        "text": "CRITICAL kernel panic risk: nvme0n1 write latency spiked to 4200ms, queue depth 64",
        "salience": 0.98,
        "source": "kernel-dmesg",
        "timestamp": "2026-09-14T20:15:00Z"
      }
    ],
    "total_chunks": 1,
    "salient_chunks": 1,
    "noise_discarded": 0,
    "reduction_rate": 0.0,
    "latency_ms": 0.95
  },
  "recall": {
    "nodes": [
      {
        "id": "ent-nvme-flush",
        "entity_type": "runbook",
        "label": "NVMe Controller Reset & Throttle Policy",
        "summary": "Execute sync flush and rate-limit queue depth to 16 if latency exceeds 2000ms.",
        "created_at": "2026-01-10T12:00:00Z",
        "last_accessed_at": "2026-09-14T20:15:00Z",
        "access_count": 54,
        "stability_score": 0.92,
        "is_archived": false,
        "score": 0.942,
        "sim_score": 0.95,
        "frequency_score": 0.90,
        "recency_score": 0.98,
        "hop_distance": 0
      }
    ],
    "edges": [],
    "query_latency_ms": 1.62
  },
  "deliberate": {
    "status": "ok",
    "step_index": 0,
    "thought": "Observed write latency of 4200ms far exceeds the 2000ms threshold specified in the NVMe Controller Reset & Throttle Policy. Recommended mitigation: flush queue and cap queue depth at 16.",
    "proposed_action": "throttle_queue_and_flush(dev='nvme0n1', max_depth=16)",
    "is_complete": true,
    "candidate_actions": [
      {
        "id": "act-nvme-01",
        "type": "kernel_parameter_tune",
        "payload": {
          "device": "nvme0n1",
          "queue_depth": 16,
          "sync": true
        },
        "committed": false,
        "created_at": "2026-09-14T20:15:01Z"
      }
    ],
    "prompt_tokens": 162,
    "completion_tokens": 44,
    "total_tokens": 206,
    "prompt_eval_rate_tps": 435.2,
    "generation_rate_tps": 52.1,
    "active_goal": "Resolve storage latency crisis",
    "trajectory_length": 1,
    "timestamp": "2026-09-14T20:15:01Z"
  },
  "consolidate": {
    "status": "consolidated",
    "trace_id": "trc-stor-0099",
    "episode_id": "ep-stor-0099",
    "session_id": "sess-stor-0099",
    "nodes_updated": 1,
    "edges_reinforced": 1,
    "entities_extracted": 1,
    "nodes_fused": 0,
    "decay_applied": true,
    "latency_ms": 2.85
  },
  "total_latency_ms": 184.2
}
```

---

## 4. Agent Analysis & Interpretation

1. **Stage 1 (Sensory Gating)**: The anomalous chunk scored a high salience of `0.98`, exceeding the default threshold (`0.45`). The attention gate completed in `0.95ms`.
2. **Stage 2 (Associative Recall)**: Retrieved the governing runbook node (`ent-nvme-flush`) ranked primarily by `sim_score: 0.95` (composite score `0.942`).
3. **Stage 3 (Scratchpad Deliberation)**: The local SLM on Node 2 evaluated the rule conditions and formulated a concrete action: `throttle_queue_and_flush(dev='nvme0n1', max_depth=16)`. The step flagged `is_complete: true`.
4. **Stage 4 (Episodic Consolidation)**: The episode was persisted to Node 1 (`episode_id: ep-stor-0099`), reinforcing the node and applying background Hebbian decay in `2.85ms`.
5. **Telemetry & Budgets**: Total latency across all four distributed nodes was `184.2ms`, well within the sub-second per-hop SLA.
