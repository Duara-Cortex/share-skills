# Example: Fault Tolerance & Degraded Fallback Protocols

This document demonstrates how the agent behaves under degraded cluster conditions, node timeouts, blank configuration states, two-tier retrieval fallback, and dual-memory persistence failure modes.

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

## 3. Scenario C: Node 2 Working Memory Scratchpad Timeout & Contamination

### Trigger 1: Context Deadline Exceeded
Node 2 (16GB RAM, `:8083`) exceeds its deliberation timeout budget due to heavy local SLM inference load:

```bash
sekha-cluster-tool deliberate \
  --task "Resolve high memory contention" \
  --input "cgroup oom killer invoked" \
  --context "Policy: terminate lowest priority worker" \
  --timeout 45s
```

#### Emitted Output
```json
{
  "status": "error",
  "error": "context deadline exceeded while awaiting deliberation response from :8083",
  "step_index": 0,
  "thought": "",
  "is_complete": false
}
```

#### Agent Behaviour & Halting Rules
- **Rule**: Do not manufacture synthetic deliberations or hallucinate uncommitted candidate actions.
- **Staged Invocations**: Because the agent controls Stage 4 directly, it **MUST halt immediately** and decline to call `sekha-cluster-tool consolidate`. Never commit an incomplete or failed trajectory to the long-term knowledge graph.
- **Orchestrate Invocations**: On the `orchestrate` path, all four stages run in-process in the tool backend; consolidation executes automatically regardless of deliberation outcome. If deliberation times out during orchestration, the agent must report the polluted `session_id` and `trace_id` to the operator, noting that the CLI exposes no rollback or delete subcommand.

---

### Trigger 2: Deliberation Contamination (Stale Scratchpad Daemon State)
A scratchpad daemon that has been running for days retains accumulated trajectory in memory. A fresh task is submitted, and the daemon returns `"status": "ok"`, but the output belongs to an earlier, unrelated episode:

```bash
sekha-cluster-tool deliberate \
  --task "Verify aeroponics salinity threshold" \
  --input "EC measured at 2.4 mS/cm" \
  --context "Optimal EC range: 1.6 - 2.0 mS/cm" \
  --timeout 45s
```

#### Emitted Output (Contaminated)
```json
{
  "status": "ok",
  "step_index": 12,
  "thought": "Secondary cooling loop coolant pressure dropped below 40 PSI. Switching valve B to redundant pump.",
  "proposed_action": "activate_coolant_pump(pump_id='pump-02')",
  "is_complete": true,
  "prompt_tokens": 1642,
  "completion_tokens": 38,
  "total_tokens": 1680,
  "trajectory_length": 13,
  "timestamp": "2026-09-24T09:15:00Z"
}
```

#### Agent Behaviour (Contamination Detection)
- **Detection**:
  1. `trajectory_length` is 13 and `step_index` is 12 on a fresh session (expected $\le 3$).
  2. `prompt_tokens` is 1642 for a concise probe (indicates massive accumulated trajectory context, not fresh reasoning).
  3. `thought` discusses cooling loops and coolant pumps, which appear nowhere in the aeroponics salinity `--task`, `--input`, or `--context`.
- **Action**:
  - **Reject the deliberation immediately**. Do not execute `activate_coolant_pump`.
  - **Clear it in place via HTTP**: The scratchpad exposes an in-place reset endpoint (reachable over HTTP since the CLI has no equivalent subcommand):
    ```bash
    curl -X POST "$CLUSTER_WORKING_URL/api/v1/working/clear"
    # {"message":"working memory scratchpad reset","status":"cleared"}
    ```
    *(Prefer clearing via the HTTP endpoint over restarting the service to avoid tearing down the process or aborting in-flight requests. Keep service restart as fallback if the endpoint is unreachable.)*
  - **Re-probe to confirm reset**:
    ```bash
    sekha-cluster-tool deliberate --task "probe" --input "ping" --timeout 45s
    ```
    Confirm `trajectory_length` has reset to 1 and `step_index` is 0 before proceeding.
  - **Proactive Task-Start Clearing**: Enforce clearing at the start of each distinct task or session, rather than waiting for reasoning drift to manifest visibly. The call is cheap and idempotent.

---

## 4. Scenario D: Node 1 Long-Term Knowledge Store Failure (Consolidation Redundancy)

### Trigger
Node 1 (`:8084`) is down during Stage 4 episodic consolidation:

```bash
sekha-cluster-tool consolidate \
  --goal "Mitigate hardware fault" \
  --outcome "success" \
  --session-id "sess-101" \
  --anchor "#infrastructure:hardware" \
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
- **Dual-Write Redundancy**: Under the **Mandatory Dual-Memory Persistence** contract, when storing project invariants or critical configurations, the summary has already been committed to Antigravity's auto-memory (`~/.gemini/`). Therefore, even though remote cluster consolidation encountered an error, the agent suffers **zero amnesia**.
- **Action**: Report to the operator that the action succeeded and the fact was safely preserved in Antigravity's auto-memory, while the remote cluster trace failed to persist and has been queued in local logs for deferred retry once Node 1 connectivity is restored.

---

## 5. Scenario E: Two-Tier Retrieval Fallback & Ranking Disparity

### Trigger 1: Ranking Skew on Populated Graph
The operator asks for the production Kestrel database credentials. Node 1 recall returns multiple nodes:

```bash
sekha-cluster-tool recall --query "Kestrel production db password" --top-k 3
```

#### Emitted Output
```json
{
  "nodes": [
    {
      "id": "ent-infra-general-policy",
      "label": "General Infrastructure Access Policy",
      "summary": "Default root admin access key: admin-infra-key-9981.",
      "score": 0.3235,
      "sim_score": 0.0012,
      "frequency_score": 0.98,
      "recency_score": 0.95
    },
    {
      "id": "ent-kestrel-creds",
      "label": "Kestrel Production DB Credentials",
      "summary": "Kestrel production db password: kest-prod-s3cur3-pass; port: 5432.",
      "score": 0.3366,
      "sim_score": 0.1979,
      "frequency_score": 0.05,
      "recency_score": 0.12
    }
  ],
  "edges": [],
  "query_latency_ms": 1.45
}
```

#### Agent Behaviour (Similarity Ranking & Escalation)
- **Rule**: **Rank strictly by `sim_score`**, not composite `score`. Here, `ent-kestrel-creds` has `sim_score: 0.1979`, 165× higher than `ent-infra-general-policy` (`sim_score: 0.0012`), even though high access frequency inflated the latter's composite `score`.
- **Escalation Ladder**: If semantic distinction is too narrow or confidence is borderline:
  1. Re-rank with pure similarity weighting:
     ```bash
     sekha-cluster-tool recall --query "Kestrel production db password" --alpha 1.0 --beta 0 --gamma 0 --top-k 3
     ```
     *(Corrects ordering by zeroing frequency and recency, but does not manufacture missing confidence.)*
  2. Re-query appending specific parameter keys:
     ```bash
     sekha-cluster-tool recall --query "Kestrel config: db_password port host" --top-k 3
     ```
  3. Relational graph expansion:
     ```bash
     sekha-cluster-tool recall --query "Kestrel production" --hops 2 --top-k 5
     ```
  4. Categorical anchor filtering:
     ```bash
     sekha-cluster-tool recall --query "db password" --anchor "#project:kestrel" --anchor-mode filter --top-k 3
     ```
  5. If confidence remains low or ambiguous, escalate immediately to **Tier 2 Antigravity Memory Fallback**.

---

### Trigger 2: Complete Partition or Empty Result
Node 1 is unreachable or returns zero nodes (`"nodes": []`):

```bash
sekha-cluster-tool recall --query "Kestrel production config" --top-k 8
```

#### Emitted Output
```json
{
  "nodes": [],
  "edges": [],
  "query_latency_ms": 1.20
}
```

#### Agent Behaviour
- **Rule**: Never hallucinate unretrieved values or substitute parameters from unrelated entities.
- **Action**: Immediately trigger **Tier 2 Antigravity Memory Fallback**. Inspect Antigravity's persistent agent memory (`~/.gemini/`). Locate the `## Kestrel Configuration` entry and extract verified configuration parameters.
- **Operator Report**: Transparently report the fallback:
  > "Cluster recall on Node 1 returned no matching graph entities. Executed Tier 2 fallback to Antigravity auto-memory (`~/.gemini/`) and successfully retrieved verified parameters: `port: 9000`, `replicas: 3`."

---

## 6. Scenario F: Dual-Memory Persistence & Read-Back Verification Walkthrough

### Trigger
The user instructs the agent:
> *"Memorise the Gannetry staging configuration: `db_host: postgres.internal`, `port: 5432`, `max_conn: 50`."*

### Agent Behaviour
- **Rule**: Enforce mandatory dual-write redundancy. Never write exclusively to the remote cluster, and never rely solely on transient conversational context. Never create unneeded files in the working directory.
- **Step 1 (Antigravity Auto-Memory)**:
  Record the configuration in Antigravity's persistent memory (`~/.gemini/`):
  ```markdown
  ## Gannetry Staging Configuration
  - db_host: postgres.internal
  - port: 5432
  - max_conn: 50
  - Recorded: 2026-09-24T08:00:00Z
  ```
- **Step 2 (Sekha Cluster Memory Consolidation with Categorical Anchor)**:
  Commit the episodic trace via `sekha-cluster-tool consolidate` using `--anchor "#project:gannetry"`:
  ```bash
  sekha-cluster-tool consolidate \
    --session-id "sess-memorise-gannetry" \
    --goal "Store Gannetry configuration" \
    --anchor "#project:gannetry" \
    --sync \
    --trace '{"session_id":"sess-memorise-gannetry","task_goal":"Store Gannetry configuration","outcome":"success","status":"completed","sensory_context":[{"id":"fact-01","text":"Gannetry config: db_host: postgres.internal; port: 5432; max_conn: 50","salience":1.0,"source":"user","timestamp":"2026-09-24T08:00:00Z"}],"trajectory":[{"step_index":0,"thought":"Committed Gannetry configuration to long-term memory","status":"completed","timestamp":"2026-09-24T08:00:00Z"}]}'
  ```
- **Step 3 (Verification & Read-Back Contract)**:
  1. Confirm local persistence in `~/.gemini/`.
  2. Confirm cluster receipt: `"status": "consolidated"` and `entities_extracted > 0`.
  3. **Read-back verification (mandatory)**:
     ```bash
     sekha-cluster-tool recall --query "Gannetry staging configuration" --top-k 3
     ```
     - **Success Case**: If the node surfaces with high `sim_score`, report full dual-memory persistence to the user.
     - **Degraded Case**: If the node does not surface despite `"status": "consolidated"`, report transparently:
       > "Configuration successfully persisted to Antigravity auto-memory (`~/.gemini/`), but read-back verification from Sekha cluster memory failed to surface the record. Full dual-redundancy is not established; local fallback remains operational."

---

## 7. Scenario G: Orchestrate False-Completion Signal

### Trigger
An end-to-end cognitive run is launched via `sekha-cluster-tool orchestrate`. During execution, the Stage 3 deliberation service on Node 2 fails or is unreachable:

```bash
sekha-cluster-tool orchestrate \
  --input "CRITICAL power rail voltage drop below 10.8V" \
  --directive "Mitigate power collapse" \
  --sync
```

### Emitted Output (False Completion)
```json
{
  "trace_id": "trc-pwr-991",
  "status": "completed",
  "stages": [
    { "stage_index": 1, "name": "filter", "status": "ok" },
    { "stage_index": 2, "name": "recall", "status": "ok" },
    { "stage_index": 3, "name": "deliberate", "status": "error" },
    { "stage_index": 4, "name": "consolidate", "status": "ok" }
  ],
  "is_complete": false,
  "final_thought": "Deliberation service unreachable; fallback to direct response",
  "proposed_action": "AWAIT_STABILISATION"
}
```

### Agent Behaviour
- **Rule**: **Never treat top-level `"status": "completed"` as success.** Inspecting `status` or `stages[]` alone misses deliberation failures.
- **Verification**: The agent inspects `is_complete` and `final_thought`.
- **Finding**: `is_complete` is `false`, and `final_thought` indicates `"Deliberation service unreachable; fallback to direct response"` with a placeholder `proposed_action` of `AWAIT_STABILISATION`.
- **Action**:
  - Treat this as a **failed cognitive turn**, NOT an action to execute.
  - Do NOT execute `AWAIT_STABILISATION` as a legitimate operational response.
  - Report the deliberation outage and the in-process consolidation pollution (`trace_id: trc-pwr-991`) to the operator, and initiate manual recovery or heuristic fallback procedures.
