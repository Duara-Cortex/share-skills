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

Run the command with a timeout of at least 600 s; a normal run takes about 45 s. The default output is concise (count-only summaries elided below), and the process exits with code `0`:

```json
{
  "status": "completed",
  "is_complete": true,
  "stages": [
    { "stage_name": "1_sensory_filter", "status": "success", "duration_ms": 95.3 },
    { "stage_name": "2_long_term_recall", "status": "success", "duration_ms": 132.8 },
    { "stage_name": "3_working_deliberate", "status": "success", "duration_ms": 13240.6 },
    { "stage_name": "4_memory_consolidate", "status": "success", "duration_ms": 16811.9 }
  ],
  "final_thought": "Observed write latency of 4200ms far exceeds the 2000ms threshold specified in the NVMe Controller Reset & Throttle Policy. Recommended mitigation: flush queue and cap queue depth at 16.",
  "proposed_action": "throttle_queue_and_flush(dev='nvme0n1', max_depth=16)",
  "trace_id": "trc-stor-0099",
  "session_id": "sess-stor-0099",
  "total_duration_ms": 30280.6,
  "loop_complete": true,
  "sensory": { "…": "count-only summary" },
  "recall": { "…": "count-only summary, including relevance_gate counts" },
  "deliberation": { "…": "count-only summary" },
  "consolidation": { "…": "count-only summary" }
}
```

The default output carries no chunk text, recalled nodes, or gate decisions. Do not pass `--full` to get them; it is for operators debugging the tool. When the salient chunk text is needed, take the staged path and run `filter --full`.

---

## 4. Agent Analysis & Interpretation

1. **Cycle Outcome**: Exit code `0`, `status` `completed` and `loop_complete` `true` — all four stages report `success`, so the cycle succeeded. (`is_complete` `true` is Node 2's own deliberation flag and is not the success signal.)
2. **Stages 1–2 (Sensory Gating & Recall)**: Both succeeded well inside their deadlines (`1_sensory_filter` 95 ms, `2_long_term_recall` 133 ms). Their summaries are count-only in the default output.
3. **Stage 3 (Scratchpad Deliberation)**: The local SLM on Node 2 formulated a concrete action: `throttle_queue_and_flush(dev='nvme0n1', max_depth=16)`, in about 13 s.
4. **Stage 4 (Episodic Consolidation)**: The episode was persisted to Node 1 in about 17 s.
5. **Telemetry & Deadlines**: `total_duration_ms` was 30280.6 (about 30 s), within the default stage deadlines; trace `trc-stor-0099`.
