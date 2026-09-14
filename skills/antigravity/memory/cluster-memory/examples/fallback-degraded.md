# Example: Fault Tolerance & Degraded Fallback Protocols

This document demonstrates how the agent behaves under degraded cluster conditions, node timeouts, and blank configuration states.

---

## 1. Scenario A: Uninitialised Configuration (Blank Endpoint)

### Trigger
An operator runs a cluster memory command on a fresh environment where endpoint URLs have not been configured in `.env` or the shell environment:

```bash
sekha-cluster-tool filter --text "kernel log line"
```

### Emitted Output (`stderr` / JSON error on `stdout`)
```json
{
  "error": "configuration error: sensory node endpoint is blank",
  "resolution": "Run 'sekha-cluster-tool env init' or configure CLUSTER_SENSORY_URL in .env"
}
```

### Agent Behaviour
- **Rule**: Never attempt to guess or hardcode private IP addresses.
- **Action**: Prompt the operator to initialise `.env` using `sekha-cluster-tool env init` or `make env` in the tool repository, and inspect configuration with `sekha-cluster-tool env show`.

---

## 2. Scenario B: Node 3 Sensory Attention Gate Outage

### Trigger
Node 3 (4GB Sensory Filter, `:8081`) is unreachable due to network partition or daemon restart:

```bash
sekha-cluster-tool filter \
  --text "Sensory probe alert: temperature exceeding 75C on Node 2"
```

### Emitted Output
```json
{
  "error": "dial tcp: connection refused",
  "fallback": "raw text preserved as unranked salient chunk",
  "chunks": [
    {
      "id": "fallback-chunk-01",
      "text": "Sensory probe alert: temperature exceeding 75C on Node 2",
      "salience": 1.0,
      "source": "fallback-heuristic",
      "timestamp": "2026-09-14T20:20:00Z"
    }
  ]
}
```

### Agent Behaviour
- **Rule**: Do not crash or abort the cognitive cycle if sensory filtering fails.
- **Action**: Treat the entire raw input as an unranked salient chunk with default unit salience (`1.0`), flagging the degraded state and proceeding directly to Stage 2 (`recall`) and Stage 3 (`deliberate`).

---

## 3. Scenario C: Node 2 Working Memory Scratchpad Timeout

### Trigger
Node 2 (16GB RAM, `:8083`) exceeds its deliberation timeout budget (default `8000ms`) due to heavy local SLM inference load:

```bash
sekha-cluster-tool deliberate \
  --task "Resolve high memory contention" \
  --input "cgroup oom killer invoked" \
  --context "Policy: terminate lowest priority worker"
```

### Emitted Output
```json
{
  "status": "error",
  "error": "context deadline exceeded while awaiting deliberation response from :8083",
  "step_index": 0,
  "thought": "",
  "is_complete": false
}
```

### Agent Behaviour
- **Rule**: Do not manufacture synthetic deliberations or hallucinate uncommitted candidate actions.
- **Action**: Mark Stage 3 as failed. If deterministic local heuristic fallback rules exist, propose a safe fallback action; otherwise, halt the cognitive loop safely and report the timeout degradation to the operator. Do not commit an unverified success to Stage 4 consolidation.

---

## 4. Scenario D: Node 1 Long-Term Knowledge Store Failure

### Trigger
Node 1 (`:8084`) is down during Stage 4 episodic consolidation:

```bash
sekha-cluster-tool consolidate \
  --goal "Mitigate hardware fault" \
  --outcome "success" \
  --session-id "sess-101" \
  --sync
```

### Emitted Output
```json
{
  "status": "error",
  "error": "failed to connect to knowledge store at :8084",
  "trace_id": "trc-pwr-20260914"
}
```

### Agent Behaviour
- **Rule**: Do not invent fake graph mutation statistics (`nodes_updated`, `edges_reinforced`).
- **Action**: Report to the operator that the deliberation action succeeded but the episodic memory trace could not be persisted. Queue the trace in local working logs for deferred retry once Node 1 connectivity is restored.
