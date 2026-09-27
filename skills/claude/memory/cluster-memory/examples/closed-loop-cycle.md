# Example — Closed-Loop Cognitive Cycle (`orchestrate`)

A single `orchestrate` call runs all four stages, threads one `X-Trace-ID`, and returns a concise result: `status`, `loop_complete`, per-stage `stages[]` and count-only stage summaries. This is the preferred path for an end-to-end turn.

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
Run it with the Bash tool's `timeout` parameter set to `600000`. A normal run takes about 45 s, and the default stage deadlines add up to about 270 s.

## Response on `stdout` (exit code 0; count-only summaries elided)
```json
{
  "status": "completed",
  "is_complete": true,
  "stages": [
    { "stage_name": "1_sensory_filter", "status": "success", "duration_ms": 120.4 },
    { "stage_name": "2_long_term_recall", "status": "success", "duration_ms": 141.9 },
    { "stage_name": "3_working_deliberate", "status": "success", "duration_ms": 12810.2 },
    { "stage_name": "4_memory_consolidate", "status": "success", "duration_ms": 16902.7 }
  ],
  "final_thought": "Core temperature 92C exceeds the 70C policy threshold; auxiliary cooling must be engaged.",
  "proposed_action": "trigger auxiliary fan override",
  "trace_id": "trc-a1b2c3d4e5f60718",
  "session_id": "sess-thermal-01",
  "total_duration_ms": 30150.0,
  "loop_complete": true,
  "sensory": { "…": "count-only summary" },
  "recall": { "…": "count-only summary, including relevance_gate counts" },
  "deliberation": { "…": "count-only summary" },
  "consolidation": { "…": "count-only summary" }
}
```

The default output carries no chunk text, recalled nodes, or gate decisions. Do not pass `--full` to get them; it is for operators debugging the tool. If you need the salient chunk text, take the staged path and run `filter --full` (see [`staged-invocation.md`](staged-invocation.md)).

## How the agent reports it
> The closed loop succeeded: exit code 0, `status` completed and `loop_complete` true, with all four stages reporting `success`. The scratchpad concluded that the 92C core exceeds the 70C threshold and proposed **trigger auxiliary fan override**. The episode was consolidated. Turn completed in `total_duration_ms` 30150; trace `trc-a1b2c3d4e5f60718`.
