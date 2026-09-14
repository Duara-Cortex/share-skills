# Example — Staged Manual Invocation (four commands)

When you need to inspect or intervene between stages — for example, to iterate deliberation or to choose recall parameters based on what the gate returned — run the four subcommands by hand instead of `orchestrate`. Each prints its stage contract to `stdout`.

## Stage 1 — Gate the stream (`filter`)
```bash
sekha-cluster-tool filter \
  --text "disk sda read latency nominal; smartd: Device: /dev/sdb, 14 Currently unreadable (pending) sectors; ntpd: adjusting clock; smartd: Device: /dev/sdb, SMART Usage Attribute: 197 Current_Pending_Sector changed from 100 to 088" \
  --directive "detect impending disk failure" \
  --threshold 0.45
```
```json
{
  "chunks": [
    { "id": "c2", "text": "smartd: Device: /dev/sdb, 14 Currently unreadable (pending) sectors", "salience": 0.88, "source": "syslog", "timestamp": "2026-09-14T10:02:11Z" },
    { "id": "c4", "text": "smartd: Device: /dev/sdb, SMART Usage Attribute: 197 Current_Pending_Sector changed from 100 to 088", "salience": 0.90, "source": "syslog", "timestamp": "2026-09-14T10:02:13Z" }
  ],
  "total_chunks": 4,
  "salient_chunks": 2,
  "noise_discarded": 2,
  "reduction_rate": 0.5,
  "latency_ms": 0.9
}
```
Carry the two `/dev/sdb` chunks forward; drop the `ntpd` and nominal-latency noise.

## Stage 2 — Ground with recall (`recall`)
```bash
sekha-cluster-tool recall --query "pending sector SMART disk failure policy" --top-k 3
```
```json
{
  "nodes": [
    { "id": "n-disk", "entity_type": "policy", "label": "Predictive Disk Replacement", "summary": "Rising Current_Pending_Sector counts warrant pre-emptive replacement and RAID rebuild.", "created_at": "2026-07-10T00:00:00Z", "last_accessed_at": "2026-09-14T10:02:13Z", "access_count": 8, "stability_score": 0.58, "is_archived": false, "score": 0.81, "sim_score": 0.79, "frequency_score": 0.30, "recency_score": 0.90, "hop_distance": 0 }
  ],
  "edges": [
    { "source_id": "n-disk", "target_id": "n-raid", "relation_type": "requires", "weight": 0.64, "created_at": "2026-07-10T00:00:00Z" }
  ],
  "query_latency_ms": 38.5
}
```
Distil to `long_term_context`: "Rising pending sectors warrant pre-emptive replacement; requires a RAID rebuild."

## Stage 3 — Deliberate (`deliberate`)
```bash
sekha-cluster-tool deliberate \
  --task "decide on disk sdb mitigation" \
  --input "sdb pending sectors rising: 100 to 088, 14 unreadable" \
  --context "Rising pending sectors warrant pre-emptive replacement; requires a RAID rebuild."
```
```json
{
  "status": "ok",
  "step_index": 0,
  "thought": "The falling normalised value and 14 pending sectors indicate imminent sdb failure; policy calls for pre-emptive replacement.",
  "proposed_action": "schedule sdb replacement and initiate RAID rebuild",
  "is_complete": true,
  "prompt_tokens": 198,
  "completion_tokens": 44,
  "total_tokens": 242,
  "prompt_eval_rate_tps": 176.0,
  "generation_rate_tps": 40.8,
  "active_goal": "decide on disk sdb mitigation",
  "trajectory_length": 1,
  "timestamp": "2026-09-14T10:02:15Z"
}
```
`is_complete` is true, so no further deliberation step is needed. (Were it false, feed the observed result back as the next `--input`.)

## Stage 4 — Consolidate (`consolidate`)
```bash
sekha-cluster-tool consolidate \
  --goal "decide on disk sdb mitigation" \
  --outcome "success" \
  --session-id "sess-disk-02" \
  --sync
```
```json
{
  "status": "consolidated",
  "trace_id": "trc-99f0aa11bb22cc33",
  "message": "Episode fused into knowledge graph.",
  "synchronous": true,
  "entities_extracted": 2,
  "nodes_fused": 1,
  "edges_reinforced": 1
}
```

## Summary reported to the operator
> Gated to 2 salient `/dev/sdb` chunks (`reduction_rate` 0.50). Recall grounded on **Predictive Disk Replacement** (`score` 0.81), which `requires` a RAID rebuild. Deliberation proposed **schedule sdb replacement and initiate RAID rebuild** (`is_complete` true). Episode consolidated under trace `trc-99f0aa11bb22cc33`.
