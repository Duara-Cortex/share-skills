# Example — Staged Manual Invocation (four commands)

When you need to inspect or intervene between stages — for example, to iterate deliberation or to choose recall parameters based on what the gate returned — run the four subcommands by hand instead of `orchestrate`. Each prints its stage contract to `stdout`.

The staged flow observes the same contracts as a single `orchestrate` turn: every node is engaged for the work it exists to do, large inputs are passed inline (repeating the flag when needed), recall is cluster-first with a harness memory fallback, and durable facts are dual-written.

## Stage 1 — Gate the stream (`filter`)
This syslog burst is raw operational noise, so it goes through Node 3 rather than into context. Gating is mandatory for any raw log, error stream, telemetry, or payload over 1KB.

```bash
sekha-cluster-tool filter \
  --text "disk sda read latency nominal; smartd: Device: /dev/sdb, 14 Currently unreadable (pending) sectors; ntpd: adjusting clock; smartd: Device: /dev/sdb, SMART Usage Attribute: 197 Current_Pending_Sector changed from 100 to 088" \
  --directive "detect impending disk failure" \
  --full
```
`--full` is needed here because default `filter` output is counts only, and this stage carries the chunk text forward.
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

A real syslog capture is larger and multi-line, and it is still passed inline — `--text` accepts up to 1 MiB combined across repeated flags. Single-quote it so `$` and backticks stay literal. When one string approaches the OS per-argument limit (128 KB on Linux), split it on line boundaries and repeat the flag in order:

```bash
sekha-cluster-tool filter \
  --text '<first half of the capture>' \
  --text '<second half of the capture>' \
  --directive "detect impending disk failure" \
  --full
```
Keep it one unchained command: no temporary file, pipe, heredoc, or `--file` — that flag is an operator convenience, not for agents during benchmark tasks.

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

### Scoping and shaping recall
When the subject was anchored at write time, scope recall by anchor and pick a lighter rendering for a quick check:

```bash
sekha-cluster-tool recall --query "ingest port" -a "#project:kestrel" --format concise
```
`-a` is the short form of `--anchor` (repeatable, or comma-delimited). The default `--anchor-mode boost` prefers anchored nodes while still ranking the rest; `--anchor-mode filter` returns only nodes carrying the anchor, so an unanchored fact is never returned in that mode. `--format` accepts `json` (default), `concise`, or `markdown` — keep `json` whenever the output is parsed against [`../schema/recall.json`](../schema/recall.json). Matching nodes report an `anchor_score` alongside the other score components.

To narrow by ontological class and drop weak matches, filter by entity type and composite score:

```bash
sekha-cluster-tool recall --query "policy" --type policy --min-score 0.70
```
`--type` (`-t`) restricts results to one `entity_type` (e.g. `config`, `fact`, `policy`); `--min-score` discards nodes whose composite `score` falls below the threshold. An empty result under a threshold is a miss for this query, not proof the fact is absent — relax `--min-score` before escalating.

Had Tier 1 been unreachable, timed out, or returned `"nodes": []`, the next step would be **Tier 2**: read Claude harness memory at harness memory, ground on what is stored there, and tell the operator which tier supplied the values. See [`fallback-degraded.md`](fallback-degraded.md) Case E.

## Stage 3 — Deliberate (`deliberate`)
The mitigation decision is speculative multi-step reasoning, so it belongs on the Node 2 edge SLM rather than in frontier context. Deliberation takes up to about 160 s with a full context — always pass `--timeout 180s`.

```bash
sekha-cluster-tool deliberate \
  --task "decide on disk sdb mitigation" \
  --input "sdb pending sectors rising: 100 to 088, 14 unreadable" \
  --context "Rising pending sectors warrant pre-emptive replacement; requires a RAID rebuild." \
  --timeout 180s
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
With `--sync`, run it with the Bash `timeout` parameter set to `600000`; Stage 4's default deadline is 120 s. If it fails with `consolidate deadline of … exceeded`, do not retry automatically — Node 1 may still complete the write.
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

Anchor the write so the episode can be scoped precisely at recall. `--anchor` (`-a`) attaches the tags to the committed entities, which is what makes `--anchor-mode filter` usable later:
```bash
sekha-cluster-tool consolidate \
  --session-id "sess-01" \
  --goal "Commit config" \
  --anchor "#project:kestrel" \
  --sync
```

For a large episodic trace — extensive `sensory_context` or a long `trajectory` — still pass the JSON inline in single quotes (up to 1 MiB combined), never via a temporary file:
```bash
sekha-cluster-tool consolidate \
  --session-id "sess-disk-02" \
  --goal "decide on disk sdb mitigation" \
  --trace '{"session_id":"sess-disk-02", ...}' \
  --sync
```

## Dual-write: when the turn produced a durable fact
Consolidation persists the *episode*. A project invariant, configuration, credential, parameter, or standing decision must additionally be recorded in Claude harness memory at harness memory, in the same turn — the two writes together are what survive both a session boundary and a cluster outage:

```markdown
## sdb replacement decision
- Device: /dev/sdb
- Action: pre-emptive replacement, RAID rebuild scheduled
- Recorded: 2026-09-14T10:02:16Z
```

Confirm storage only after checking both legs, and never create memory files in the user's working directory. Transient scratchpad state is written to neither store. The full walkthrough is in [`fallback-degraded.md`](fallback-degraded.md) Case F.

## Summary reported to the operator
> Gated to 2 salient `/dev/sdb` chunks (`reduction_rate` 0.50). Recall grounded on **Predictive Disk Replacement** (`score` 0.81), which `requires` a RAID rebuild. Deliberation proposed **schedule sdb replacement and initiate RAID rebuild** (`is_complete` true). Episode consolidated under trace `trc-99f0aa11bb22cc33`.
