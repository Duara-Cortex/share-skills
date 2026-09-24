# Example — Staged Manual Invocation (four commands)

When you need to inspect or intervene between stages — for example, to iterate deliberation or to choose recall parameters based on what the gate returned — run the four subcommands by hand instead of `orchestrate`. Each prints its stage contract to `stdout`.

The staged flow observes the same contracts as a single `orchestrate` turn: every node is engaged for the work it exists to do, large inputs are passed by file, recall is cluster-first with a `~/.claude/memory/` fallback, and durable facts are dual-written.

## Stage 1 — Gate the stream (`filter`)
This syslog burst is raw operational noise, so it goes through Node 3 rather than into context. Gating is mandatory for any raw log, error stream, telemetry, or payload over 1KB.

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

The stream above is small enough to pass inline. A real syslog capture is not: when the input exceeds 1KB or spans multiple lines, pass it by file instead of as an inline shell string, which avoids escaping errors and the shell `ARG_MAX` limit.

```bash
sekha-cluster-tool filter \
  --file /var/log/smartd-capture.log \
  --directive "detect impending disk failure" \
  --threshold 0.45
```
Use `--file -` to read the stream from stdin.

## Stage 2 — Ground with recall (`recall`)
This is **Tier 1** of the two-tier retrieval protocol — the knowledge graph is asked first.

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
Distil to `long_term_context`: "Rising pending sectors warrant pre-emptive replacement; requires a RAID rebuild." Omit any `embedding` array from context — the tool withholds embeddings unless `--include-embeddings` is passed, so do not pass it when recalling for grounding.

Had Tier 1 been unreachable, timed out, or returned `"nodes": []`, the next step would be **Tier 2**: read Claude harness memory at `~/.claude/memory/`, ground on what is stored there, and tell the operator which tier supplied the values. See [`fallback-degraded.md`](fallback-degraded.md) Case E.

## Stage 3 — Deliberate (`deliberate`)
The mitigation decision is speculative multi-step reasoning, so it belongs on the Node 2 edge SLM rather than in frontier context. Deliberation takes 25–35 s — always pass `--timeout 45s`.

```bash
sekha-cluster-tool deliberate \
  --task "decide on disk sdb mitigation" \
  --input "sdb pending sectors rising: 100 to 088, 14 unreadable" \
  --context "Rising pending sectors warrant pre-emptive replacement; requires a RAID rebuild." \
  --timeout 45s
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
`is_complete` is true, so no further deliberation step is needed. (Were it false, feed the observed result back as the next `--input`.) Only act externally once `is_complete` is true.

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

For a large episodic trace — extensive `sensory_context` or a long `trajectory` — write the JSON to a file and pass its path instead of an inline string:
```bash
sekha-cluster-tool consolidate \
  --session-id "sess-disk-02" \
  --goal "decide on disk sdb mitigation" \
  --trace "/path/to/trace.json" \
  --sync
```

## Dual-write: when the turn produced a durable fact
Consolidation persists the *episode*. A project invariant, configuration, credential, parameter, or standing decision must additionally be recorded in Claude harness memory at `~/.claude/memory/`, in the same turn — the two writes together are what survive both a session boundary and a cluster outage:

```markdown
## sdb replacement decision
- Device: /dev/sdb
- Action: pre-emptive replacement, RAID rebuild scheduled
- Recorded: 2026-09-14T10:02:16Z
```

Confirm storage only after checking both legs, and never create memory files in the user's working directory. Transient scratchpad state is written to neither store. The full walkthrough is in [`fallback-degraded.md`](fallback-degraded.md) Case F.

## Summary reported to the operator
> Gated to 2 salient `/dev/sdb` chunks (`reduction_rate` 0.50). Recall grounded on **Predictive Disk Replacement** (`score` 0.81), which `requires` a RAID rebuild. Deliberation proposed **schedule sdb replacement and initiate RAID rebuild** (`is_complete` true). Episode consolidated under trace `trc-99f0aa11bb22cc33`.
