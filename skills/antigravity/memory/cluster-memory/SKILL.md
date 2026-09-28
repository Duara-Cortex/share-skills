---
name: cluster-memory
description: >-
  Coordinates the Tri-Node Edge Cognitive Cluster for persistent cross-session memory, sensory attention gating (Node 3), associative knowledge recall (Node 1), working scratchpad deliberation (Node 2), and episodic consolidation (Node 1). Enforces mandatory dual-memory persistence (dual-write to Antigravity harness auto-memory and Sekha cluster) and a two-tier retrieval protocol (cluster-first with Antigravity memory fallback). Use when asked to memorise, memorize, remember, save, or store any information for later use, when retrieving grounded facts across clean sessions, or when processing edge telemetry streams via sekha-cluster-tool.
---

# Cluster Memory Skill

The **Cluster Memory Skill** governs distributed cognitive workflows across a dedicated tri-node edge computing cluster. It interfaces exclusively through the compiled Go CLI harness [`sekha-cluster-tool`](https://github.com/Duara-Cortex/sekha-cluster-tool) to eliminate behavioural drift, enforce dual-memory persistence redundancy, and guarantee strict contract compliance.

---

## 🏛️ Cognitive Topology & Architecture

The architecture coordinates a **Dual-Memory Persistence Hierarchy** coupled to a **Tripartite Cognitive Model**:

1. **Dual-Memory Persistence Hierarchy**:
   - **Antigravity Auto-Memory Tier**: Antigravity's persistent agent memory provides immediate, offline-resilient, deterministic state persistence directly within the agent harness (resolved via the Harness Auto-Memory Resolution Rule; never the user's working directory or repository).
   - **Sekha Remote Cluster Tier**: The tri-node edge cluster provides high-dimensional associative knowledge graph storage, relational edge linking, and episodic consolidation.

2. **Tripartite Edge Processing Nodes**:
   - **Node 3 &bull; Sensory Attention Gate** (`:8081`): High-throughput salience filter and CPU entropy gate for noise reduction ($< 1\text{ ms}$).
   - **Node 2 &bull; Working Scratchpad** (`:8083`): Dedicated local Small Language Model (SLM) inference engine for multi-step hypothesis deliberation.
   - **Node 1 &bull; Knowledge Graph & Episodic Store** (`:8084`): Persistent relational graph, vector indices, Hebbian reinforcement, and episodic decay daemon.

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Frontier Agent
    participant Local as Antigravity Memory<br/>(Harness Memory)
    participant N3 as Node 3: Sensory Gate<br/>(:8081)
    participant N1 as Node 1: Knowledge Graph<br/>(:8084)
    participant N2 as Node 2: SLM Scratchpad<br/>(:8083)

    Note over Agent,Local: Mandatory Dual-Write (Facts & Invariants)
    Agent->>Local: 1. Commit summary to Antigravity memory
    Agent->>N1: 2. consolidate --sync --trace '{...}'
    N1-->>Agent: Confirm status: consolidated & entities_extracted > 0

    Note over Agent,Local: Two-Tier Retrieval Protocol
    Agent->>N1: Tier 1: recall --query "<subject>"
    alt Tier 1 Succeeded
        N1-->>Agent: Subgraph nodes & edges
    else Tier 1 Failed / Unreachable / Empty
        Agent->>Local: Tier 2: Inspect Antigravity memory fallback
        Local-->>Agent: Offline grounded facts
    end

    Note over Agent,N2: Tripartite Operational Cycle
    Agent->>N3: filter --text stream (>1KB payload)
    N3-->>Agent: Salient chunks (noise discarded)
    Agent->>N1: recall --query "<salient concept>"
    N1-->>Agent: Grounding context & policies
    Agent->>N2: deliberate --task ... --input ... --context ...
    N2-->>Agent: thought, proposed_action, is_complete
    Agent->>N1: consolidate --sync (commit completed trajectory)
```

### Physical Node Allocation
- **Node 3** (`4GB RAM`): Hosts the in-memory ring buffer daemon and sensory attention gate on port `:8081`.
- **Node 2** (`16GB RAM`): Hosts the working memory scratchpad and local SLM inference engine on port `:8083`.
- **Node 1** (`8GB RAM`): Hosts the persistent knowledge graph store, vector indices, and consolidation daemon on port `:8084`.

---

## 📋 Prerequisites & Tooling

1. **Compiled Binary**: `sekha-cluster-tool` ($\ge \text{v1.0.12}$) must be installed and executable in the system `PATH`. The `v1.0.12` floor is required for stateless deliberation (since v1.0.9), for large inline payloads (`--text`/`--input`/`--trace`, up to 1 MiB combined) and repeatable `--text`/`--input` flags (v1.0.10), and for the concise `orchestrate` output, truthful `status` and `loop_complete`, exit codes and configurable stage deadlines (v1.0.12).
   - Verify the installed version is v1.0.12 or later:
     ```bash
     sekha-cluster-tool --version
     ```
   - Release installer:
     ```bash
     curl -fsSL https://raw.githubusercontent.com/Duara-Cortex/sekha-cluster-tool/main/scripts/install.sh | bash
     ```

2. **Decoupled Configuration**:
   - Zero IP addresses are hardcoded in the codebase, skill files, or agent output.
   - Endpoint addresses default to blank (`""`) and must be supplied via local environment configuration.
   - Initialise a starter `.env` file when configuring a new environment:
     ```bash
     sekha-cluster-tool env init
     ```
   - Inspect resolved configuration and precedence:
     ```bash
     sekha-cluster-tool env show
     ```

### Environment Configuration Variables
```env
# Sensory Attention Layer (Node 3)
CLUSTER_SENSORY_URL=http://<sensory-node>:8081

# Working Memory Scratchpad Layer (Node 2)
CLUSTER_WORKING_URL=http://<working-node>:8083

# Long-Term Knowledge Graph Layer (Node 1)
CLUSTER_KNOWLEDGE_URL=http://<knowledge-node>:8084

# Stage Deadlines (milliseconds; tool defaults shown)
CLUSTER_SENSORY_TIMEOUT_MS=30000
CLUSTER_DEFAULT_TIMEOUT_MS=1500
CLUSTER_CONSOLIDATE_TIMEOUT_MS=120000
# CLUSTER_DELIBERATE_TIMEOUT_MS is a floor for Stage 3; see Stage Deadlines below

# Attention & Recall Hyperparameters
CLUSTER_SALIENCE_THRESHOLD=0.75
CLUSTER_RECALL_TOP_K=5
```

Configuration resolution adheres strictly to 12-factor standard precedence:
1. Explicit CLI Flags (`--sensory-url`, `--working-url`, `--knowledge-url`)
2. Operating System Environment Variables
3. Local `.env` file (loaded from working directory, `--env-file`, or `~/.config/sekha-cluster-tool/.env`)
4. Compile-time injected builder defaults
5. Blank fallback (`""`)

### Stage Deadlines
Each `orchestrate` stage has its own deadline, resolved **flag > env > `.env` > default**:

| Stage | Setting | Default |
| :--- | :--- | :--- |
| 1. Sensory | `--sensory-timeout`, `CLUSTER_SENSORY_TIMEOUT_MS` | 30 s |
| 2. Recall | `--timeout`, `CLUSTER_DEFAULT_TIMEOUT_MS` | 1.5 s |
| 3. Deliberate | `CLUSTER_DELIBERATE_TIMEOUT_MS` — a floor; the tool widens it for the packed prompt size | up to about 180 s (Node 2's own inference limit) |
| 4. Consolidate | `--consolidate-timeout`, `CLUSTER_CONSOLIDATE_TIMEOUT_MS` | 120 s |

On `orchestrate`, `--timeout` governs **recall only**. A deadline error names the value used and where it came from. A discrete `deliberate` call takes `--timeout 180s` (45 s for the preflight probe only). Measured live on v1.0.13 with `--sync`: on a ~90 KB input, sensory 0.1–0.3 s, recall 0.1–0.3 s, deliberate 120–160 s (Node 2 now reads a full context), consolidate 0.5–9 s — about 2–3 minutes in total; a one-line input takes 15–40 s.

---

## 💾 Mandatory Dual-Memory Persistence (Dual-Write Contract)

Writing facts exclusively to the remote cluster introduces a fatal single point of failure: network partitions or daemon maintenance cause total amnesia. To guarantee offline survivability and distributed associative recall, the agent **MUST enforce dual-write redundancy**.

### The Dual-Write Mandate
Whenever instructed to **memorise (memorize)**, **store**, **remember**, or **record** critical project facts, configurations, credentials, or architecture invariants, the agent **MUST execute two coordinated writes**:

1. **Write to Antigravity auto-memory**:
   - Commit the structured key-value summary directly to Antigravity's persistent agent harness memory (resolved via the Harness Auto-Memory Resolution Rule). Never create extraneous memory files in the user's working directory or repository.
2. **Write to Sekha cluster memory**:
   - Commit the episodic trace via `sekha-cluster-tool consolidate --sync --trace '...'`.

### Harness Auto-Memory Resolution Rule
Before reading or writing local harness memory, the agent must resolve the target directory using the following preference order:
1. **Harness App Data Memory (Primary & Most Specific)**: `~/.gemini/antigravity-cli/memory/` (may be exposed via `$APP_DATA_DIR/memory/` or harness configuration).
2. **Global Gemini Memory Fallback**: `~/.gemini/memory/`.

> [!IMPORTANT]
> **Working-Directory Boundary & Preflight Check**:
> - The agent **MUST check which directory actually exists before writing**, because a write to a path the harness never reads back is silently lost.
> - **Never the user's working directory or repository**: Under no circumstances may the agent fall back to writing memory files, notes, or scratchpads into the active project workspace or repository.
> - **Missing Candidate Action**: If neither candidate directory exists or the location cannot be determined, do not invent a location; the agent must halt and ask the operator where harness memory lives.

### State Classification & Trigger Rules

| State Classification | Definition & Scope | Persistence Action |
| :--- | :--- | :--- |
| **Transient Scratchpad State** | Ephemeral loop counters, intermediate tool output, draft reasoning thoughts, temporary debug logs, uncommitted candidate actions. | **Local session only.** Do NOT write to long-term memory or Sekha cluster consolidation. |
| **Project Invariants & Grounding Facts** | Architectural constants, project configurations, service credentials, environment parameters, operational policies, permanent system constraints, cross-session user decisions. | **Mandatory Dual-Write.** MUST be committed to Antigravity auto-memory (resolved harness directory; never the user's working directory or repository) AND consolidated to Sekha cluster memory. |

### Dual-Write Execution Pattern

#### Step 1: Commit to Antigravity Auto-Memory
Store the invariant in Antigravity's resolved agent auto-memory directory (never the user's working directory or repository):
```markdown
## <Subject> Configuration
- <PARAM_1>: <value_1>
- <PARAM_2>: <value_2>
- Recorded: <now UTC>
```

#### Step 2: Commit to Sekha Cluster Memory
Execute the single-line inline consolidation command (no heredocs, subshells, or temporary files):
```bash
sekha-cluster-tool consolidate \
  --session-id "sess-memorise-<subject>" \
  --goal "Store <Subject> configuration" \
  --anchor "#project:<subject>" \
  --sync \
  --trace '{"session_id":"sess-memorise-<subject>","task_goal":"Store <Subject> configuration","outcome":"success","status":"completed","sensory_context":[{"id":"fact-01","text":"<Subject> config: <PARAM_1>: <val>; <PARAM_2>: <val>; ...","salience":1.0,"source":"user","timestamp":"<now UTC>"}],"trajectory":[{"step_index":0,"thought":"Committed <Subject> configuration to long-term memory","status":"completed","timestamp":"<now UTC>"}]}'
```

> [!IMPORTANT]
> **Categorical Anchoring**: You **MUST** specify `--anchor "#project:<subject>"` during consolidation. An unanchored write is reachable only by similarity ranking and can never be returned by `--anchor-mode filter`. If an anchor is omitted at write time, downstream queries using `--anchor "#project:<subject>" --anchor-mode filter` will fail to isolate the entity in the graph.

#### Step 3: Verification & Read-Back Contract
The agent must verify that:
1. **Auto-memory written**: The Antigravity auto-memory entry has been recorded in the resolved agent harness directory (never the user's working directory or repository).
2. **Cluster write confirmed**: The cluster tool response confirms `"status": "consolidated"` and `entities_extracted > 0`. (Note: `"status": "consolidated"` and `entities_extracted > 0` confirm that the write was processed, but do NOT guarantee associative recallability.)
3. **Read-back verification (mandatory)**: Run a recall query for a distinctive phrase:
   ```bash
   sekha-cluster-tool recall --query "<distinctive subject phrase>" --top-k 3
   ```
   Confirm that the expected entity surfaces with sufficient `sim_score`. If it does not surface, report the fact as persisted to Antigravity auto-memory (resolved harness directory; never the user's working directory or repository) but not reliably retrievable from the cluster &mdash; **do not claim full dual-redundancy**.

> [!WARNING]
> **Plaintext Credential Notice**: If storing credentials, tokens, or private endpoints, explicitly inform the operator that credentials reside in plaintext within the agent auto-memory and the Node 1 knowledge graph store.

---

## 🔍 Two-Tier Retrieval Protocol (Cluster-First with Local Fallback)

When recalling facts, configurations, or operational guidelines across sessions, agents must follow a strict two-tier retrieval hierarchy:

```
[Recall Trigger] ──► Tier 1: Sekha Cluster Recall (:8084)
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
      [Hit with Valid Score]      [Empty / Error / Unreachable]
              │                           │
              ▼                           ▼
      [Ground Reasoning]          Tier 2: Antigravity Memory Fallback
                                          │
                                  ┌───────┴───────┐
                                  ▼               ▼
                            [Found in Memory]  [Not Found]
                                  │               │
                                  ▼               ▼
                          [Ground Reasoning]  [Decline Honestly]
```

### Tier 1: Sekha Cluster Recall (Primary)
1. **Query Execution**:
   ```bash
   sekha-cluster-tool recall --query "<subject name>" --top-k 8
   ```
2. **Entity Extraction & Similarity Ranking**:
   - Inspect the `nodes` array for matching `sensory_fact` nodes.
   - **Rank strictly by `sim_score`**, NOT by composite `score`. Composite `score` combines vector similarity ($\alpha$) with access frequency ($\beta$) and recency ($\gamma$). On a populated graph, heavily accessed or recent unrelated nodes (such as credentials, admin keys, or system policies) can easily outscore a verbatim match with low access counts.
   - **Forbid grounding on near-zero or null `sim_score`**: Never ground on a top-ranked node whose `sim_score` is near zero, `null`, or absent. A frequently-accessed entity outranking a verbatim match on composite `score` is the primary signature of ranking skew; treating the highest composite `score` as the best match in this state is a critical error.
   - Extract parameters verbatim from the `summary` field (never rely on truncated `label` fields).
3. **Retrieval Escalation Ladder**:
   If the target subject does not surface with high `sim_score`, or if matches are weak/ambiguous, work the escalation ladder systematically before declaring a miss or falling back to Tier 2:
   - **Step 1 (Pure Similarity Re-ranking)**: Override frequency/recency weighting to isolate semantic vector match:
     ```bash
     sekha-cluster-tool recall --query "<subject name>" --alpha 1.0 --beta 0 --gamma 0 --top-k 5
     ```
     *(Note: Pure-similarity re-ranking with `--alpha 1.0` is an escalation fallback, not a new default. It fixes ordering by stripping frequency and recency bias, but does not improve underlying semantic confidence.)*
   - **Step 2 (Key Parameter Re-query)**: Re-query by appending expected parameter keys to the query string:
     ```bash
     sekha-cluster-tool recall --query "<Subject> config: <PARAM_1> <PARAM_2>" --top-k 5
     ```
   - **Step 3 (Relational Expansion)**: Expand traversal depth across connected edges:
     ```bash
     sekha-cluster-tool recall --query "<subject name>" --hops 2 --top-k 8
     ```
   - **Step 4 (Categorical Anchor Filtering)**: Filter explicitly by anchor tag:
     ```bash
     sekha-cluster-tool recall --query "<subject name>" --anchor "#project:<subject>" --anchor-mode filter --top-k 5
     ```
   - **Step 5 (Escalate to Tier 2)**: If all steps fail or yield low confidence, immediately escalate to **Tier 2 Antigravity Memory Fallback** (resolved harness memory; never the user's working directory or repository).
   - **Anti-Pattern Guard**: **Never report "not found" directly from a default-weighted query that returned weak matches.** You must work through the escalation ladder before declaring a Tier 1 miss.
4. **Token Hygiene**:
   - Strip the dense 64-D float `"embedding"` array when assembling the working context.

### Tier 2: Antigravity Memory Fallback (Secondary)
1. **Fallback Triggers**:
   - Tier 1 encounters a connection failure, timeout, or unreachable endpoint (`dial tcp: connection refused`, i/o timeout).
   - Tier 1 returns zero matches (`"nodes": []`) or entities unrelated to the target subject.
   - Endpoint configuration is blank or reports service degradation.
2. **Fallback Inspection**:
   - Immediately inspect Antigravity's persistent agent harness memory (resolved via the Auto-Memory Resolution Rule; never the user's working directory or repository).
   - Extract the grounded parameters directly from the agent harness memory.
   - Transparently notify the operator that cluster recall was bypassed or degraded and values were recovered from Antigravity auto-memory.
3. **Strict Grounding Rule**:
   - Ground decisions **exclusively** in verified facts retrieved from Tier 1 or Tier 2.
   - **Never hallucinate unretrieved values** or substitute parameters from unrelated entities. If neither tier holds the fact, decline honestly.

---

## ⚙️ Tripartite Operational Directives & Commands

Agents must actively engage all three physical nodes in accordance with their architectural purpose, rather than treating Sekha merely as a key-value store for Node 1.

### 1. Node 3: Sensory Attention Gating (`filter`)

> [!IMPORTANT]
> **Sensory Gating Mandate**: You **MUST** invoke `sekha-cluster-tool filter` before reading raw logs, error streams, sensor telemetry, or verbose API payloads exceeding 1KB into context. Never ingest raw, un-filtered bursts (>1KB) directly into frontier LLM context.

- **Purpose**: Discards routine keepalive noise, isolates anomalous chunks, and bounds prompt token usage.
- **Invocation**:
  ```bash
  sekha-cluster-tool filter \
    --text '<raw telemetry stream>' \
    --directive "Identify operational anomalies" \
    --full
  ```
- **Rules**:
  - Pass `--full`: default `filter` output is counts only, and this stage needs the chunk text to carry forward. This is the only place the skill uses `--full` by default.
  - Do not pass `--threshold`: the cluster's configured salience threshold (`CLUSTER_SALIENCE_THRESHOLD`) applies.
  - Carry forward only the returned salient `chunks`; discard background noise.
  - Report `reduction_rate`, `noise_discarded`, and `latency_ms` to the operator.
  - If `salient_chunks` is 0, conclude explicitly that no signal exceeded the salience threshold; **never manufacture artificial signal**.
- *Schema details: [schema/filter.json](./schema/filter.json)*

### 2. Node 2: Working Memory Deliberation (`deliberate`)

> [!IMPORTANT]
> **Working Scratchpad Mandate**: You **MUST** invoke `sekha-cluster-tool deliberate` for speculative multi-step hypothesis evaluation, diagnostic root-cause analysis, or tactical decision trees before executing actions. Offload speculative planning to the local edge SLM on Node 2 rather than consuming cloud frontier LLM tokens.

- **Purpose**: Evaluates candidate actions and reasons over operational policies on the local SLM. Each call is stateless: an independent session built only from the `--task`, `--input` and `--context` supplied, with no server-side state kept between calls (`step_index` and `trajectory_length` are always 1).
- **Invocation**:
  ```bash
  sekha-cluster-tool deliberate \
    --task "<task objective>" \
    --input "<sensory observation>" \
    --context "<retrieved facts and policy guidelines>" \
    --timeout 180s
  ```
- **Rules**:
  - Edge SLM inference on Node 2 takes up to about 160 s with a full context; **always pass `--timeout 180s`** to avoid premature timeout failures (45 s only for the preflight probe).
  - **Per-Call Sanity Check**: Ensure `status == "ok"` and that the returned `thought` and `proposed_action` directly address the `--task` and `--input` supplied. If they do not, treat the call as failed, do not relay its `thought` or `proposed_action` as a finding, and state that no valid deliberation was obtained.
  - Evaluate `thought`, `proposed_action`, and `is_complete`.
  - **Caller-Owned Multi-Turn Deliberation**: Node 2 remembers nothing between calls, so a chain of reasoning exists only if you carry it. If `is_complete` is `false`, iterate by passing the previous step's `proposed_action` (and any result of acting on it) as the next call's `--input`, or inside `--context` alongside the grounding facts.
  - Only execute external actions once deliberation reaches terminal completion (`is_complete: true`).
- *Schema details: [schema/deliberate.json](./schema/deliberate.json)*

### 3. Node 1: Long-Term Knowledge Graph (`recall` & `consolidate`)

- **Associative Recall (`recall`)**:
  - Query the long-term relational graph on Node 1 for grounding policies, topological dependencies, and past episode resolutions:
    ```bash
    sekha-cluster-tool recall \
      --query "<salient concept or entity>" \
      --top-k 5
    ```
  - **Rank strictly by `sim_score`** (never composite `score`). **Forbid grounding on a top-ranked node whose `sim_score` is near zero, `null`, or absent** (a frequently-accessed node outranking a verbatim match on composite `score` indicates ranking skew). Evaluate relational `edges` to understand system topology. Follow the Retrieval Escalation Ladder if ranking is ambiguous or `sim_score` is weak.
  - *Schema details: [schema/recall.json](./schema/recall.json)*

- **Episodic Consolidation (`consolidate`)**:
  - Commit completed reasoning trajectories, actions, and outcomes to Node 1 for graph fusion, Hebbian reinforcement, and background decay:
    ```bash
    sekha-cluster-tool consolidate \
      --goal "<task objective>" \
      --outcome "success" \
      --session-id "<session id>" \
      --anchor "#project:<subject>" \
      --sync
    ```
  - With `--sync`, give the command a timeout of at least 600 s (Stage 4's default deadline is 120 s). Never retry a `consolidate deadline of … exceeded` error automatically: Node 1 may still complete the write, so a retry can store the memory twice.
  - **Categorical Anchoring**: Always pass `--anchor "#project:<subject>"` on `consolidate` so the fact can be scoped precisely at recall. An unanchored write is reachable only by similarity ranking and can never be returned by `--anchor-mode filter`.
  - For configuration memorisation, **always supply the full `--trace` JSON** per the Dual-Write Contract.
  - *Schema details: [schema/consolidate.json](./schema/consolidate.json)*

### 4. Unified Closed-Loop Pipeline (`orchestrate`)

- **Directive**: Use `sekha-cluster-tool orchestrate` whenever an end-to-end cognitive turn (Stream $\to$ Filter $\to$ Recall $\to$ Deliberate $\to$ Consolidate) is required in one shot.
- **Invocation**:
  ```bash
  sekha-cluster-tool orchestrate \
    --input '<raw sensory stream>' \
    --directive "<attention directive>" \
    --anchor "#project:<subject>" \
    --trace-id "<trace id>" \
    --sync
  ```
- **Command Timeout**: A one-line input takes 15–40 s and a ~90 KB input about 2–3 minutes; the default stage deadlines add up to about 330 s (30 + 1.5 + up to 180 + 120). Give every `orchestrate` (and `consolidate --sync`) command a timeout of at least **600 s** in the agent's command runner. If the runner kills the process, no JSON is printed and every stage outcome is lost. Keep 600 s as the ceiling: do not pass a `--consolidate-timeout` that would push the run past it. Set it in the runner, never with a shell `timeout` wrapper, which breaks the single `sekha-cluster-tool` command form.
- **Reading the Result**: `stdout` always carries exactly one JSON object; diagnostics go to `stderr`. Never merge them with `2>&1`. Judge the cycle on `status`, `loop_complete`, `stages[]` and the exit code together:

  | Exit code | Meaning | What to do |
  | :--- | :--- | :--- |
  | `0` | `status` is `completed`: no stage failed | Success, provided `loop_complete` is also `true`. |
  | `2` | `status` is `partial` (some stages failed) or `failed` (all failed) | Not a crash. Read the JSON on `stdout`, name each failed stage and quote its `error`. |
  | `1` | Error shape `{"status":"error","error":…,"trace_id":…}`; the cycle never ran | Report `error` as a configuration or invocation problem. |
  | `0` with `loop_complete` `false` | No stage `failed`, but not all reported `success` — e.g. stage 3 `over_budget` | Not a success. Name the stage that did not succeed and report it; do not retry an `over_budget` stage. |

  A cycle succeeded **only** when `status == "completed"`, `loop_complete == true` **and** the exit code is `0`. `loop_complete` is true only when all four stages report `success`. `is_complete` is Node 2's own deliberation flag and does **not** mean the loop completed.
- **Stage Telemetry**: `stages[]` lists `1_sensory_filter`, `2_long_term_recall`, `3_working_deliberate` and `4_memory_consolidate`, each with `stage_name`, `status`, `duration_ms`, and `error` on failure (capped at about 300 bytes). `status` is `success` or `failed`; stage 3 may also be `over_budget`, meaning it ran but the prompt was over the Node 2 context budget.
- **Retry Rules**:
  - **Stages 1–3, transient failure** (connection refused, HTTP 5xx, a sensory or recall deadline): retry **once** at most. A retried `orchestrate` re-runs every stage, including consolidation — if `4_memory_consolidate` already reported `success`, do not re-run the whole cycle, because that stores the episode twice; report the failure instead.
  - **Stage 3 context budget** (`over_budget`, or an error saying the prompt "would exceed the Node 2 context budget"): not transient. Report it; do not retry.
  - **Stage 4 deadline** (`consolidate deadline of … exceeded`): **never retry automatically.** Node 1 may still finish the write after the client gives up. Report it and suggest raising `CLUSTER_CONSOLIDATE_TIMEOUT_MS`, or retrying once with `--consolidate-timeout` only if the user agrees.
- **Concise Default Output**: About 2–3 KB for any input up to the 1 MiB cap. After the top-level fields and `stages[]` come count-only `sensory`, `recall` (including `relevance_gate` counts), `deliberation` and `consolidation` summaries, with no chunk text, recalled node list, or per-node gate decisions. **Never pass `--full` to `orchestrate`** unless the user explicitly asks for debug output; it restores the old full payload (about 200 KB for an 87 KB input).
- **Rules**:
  - Automatically correlates all 4 stages under a single distributed `X-Trace-ID`.
  - **Treat placeholder action as failed turn**: A fallback `final_thought` (such as `"Deliberation service unreachable; fallback to direct response"`) accompanied by a placeholder `proposed_action` (such as `AWAIT_STABILISATION`) represents a **failed turn**, reported as `status` `partial` with `3_working_deliberate` failed. Never report turn success, and never present `AWAIT_STABILISATION` as an action to carry out or schedule.
  - **Consolidation Consequence**: Because all 4 stages execute in-process within the tool backend, Stage 4 (`consolidate`) commits automatically even when Stage 3 deliberation fails or times out. A degraded episode with placeholder reasoning has entered the knowledge graph under the active `session_id`/`trace_id` and will surface in future recalls. Note that the CLI exposes no delete, rollback, prune, or archive subcommand, so there is no remediation path through the tool.
  - **Deliberation Deadline**: On `orchestrate`, `--timeout` governs recall only; it neither caps the whole run nor extends Stage 3. Stage 3's deadline comes from `CLUSTER_DELIBERATE_TIMEOUT_MS`, which the tool widens to fit the prompt (see **Stage Deadlines**).
- *Schema details: [schema/orchestrate.json](./schema/orchestrate.json)*

### 5. Large Payload & Stream Handling Protocol (Inline, Repeatable Flags)

> [!TIP]
> **Inline Payload Delivery**: `--text`, `--input`, and `--trace` accept large inline strings — up to 1 MiB combined across repeated flags (`--max-input-bytes`/`CLUSTER_MAX_INPUT_BYTES`); over the cap the tool exits `1` and never truncates (`sekha-cluster-tool >= v1.0.10`) —, including multi-line content. Pass every payload inline in **one unchained `sekha-cluster-tool` command**, and never write it to a temporary file.

- **Single Inline Value**:
  ```bash
  sekha-cluster-tool filter \
    --text '<raw sensory stream>' \
    --directive "Isolate anomalous operational events" \
    --full
  ```
- **Repeatable Flags for Very Large Payloads**:
  When a single string approaches the operating system's argument limits (Linux caps one argument at 128 KB, whatever the tool accepts), split the payload on line boundaries and pass each chunk, in order, as its own flag:
  ```bash
  sekha-cluster-tool orchestrate \
    --directive "Analyse telemetry and formulate mitigation" \
    --input '<chunk 1>' \
    --input '<chunk 2>' \
    --sync
  ```
- **Large Inline Trace Consolidation (`consolidate --trace`)**:
  Pass a large deliberation trace inline as single-quoted JSON (up to 1 MiB combined):
  ```bash
  sekha-cluster-tool consolidate \
    --session-id "sess-large-trace" \
    --goal "Consolidate complex diagnostic episode" \
    --trace '{"session_id":"sess-large-trace", ...}' \
    --sync
  ```
- **Quoting**: Wrap payloads in single quotes so `$`, backticks, and `!` stay literal; write an embedded single quote as `'\''`.
- **Forbidden Delivery Routes**: Temporary files, pipes (`cat ... |`), heredocs, redirection, shell variables, and chained commands (`&&`, `;`). Benchmark sessions permit exactly one unchained `sekha-cluster-tool` command with no temporary-file writes, so any of these voids the session.

> [!IMPORTANT]
> **`--file` is an operator convenience only.** The tool's `--file <path>` / `--file -` and `--trace <path>` forms exist for humans running the CLI by hand. Agents must **NOT** use them during benchmark tasks.

### 6. Cluster Health & Deliberation Preflight Probe (`status` & probe)

- **Network Reachability Probe (`status`)**:
  Probe reachability and measure roundtrip network latency across all three endpoints before critical runs:
  ```bash
  sekha-cluster-tool status
  ```
- **Deliberation Preflight Probe (Health ≠ Working)**:
  `status` probes only `/api/v1/memory/health` and `/api/v1/working/health`, which check daemon HTTP liveness and report `all_nodes_healthy` even when SLM inference is deadlocked or saturated. Before launching complex multi-turn workflows or expensive orchestrations:
  1. Execute a deliberation probe:
     ```bash
     sekha-cluster-tool deliberate --task "probe" --input "ping" --timeout 45s
     ```
  2. Verify that the probe passes the Per-Call Sanity Check: `"status": "ok"`, with a `thought` and `proposed_action` that directly address the `--task` and `--input` supplied.

---

## 📖 Progressive Disclosure & Reference Links

- **Payload Contracts**:
  - [Sensory Filter Schema](./schema/filter.json) &mdash; Request and response specifications for Stage 1.
  - [Associative Recall Schema](./schema/recall.json) &mdash; Knowledge graph query and subgraph response for Stage 2.
  - [Working Deliberation Schema](./schema/deliberate.json) &mdash; Scratchpad reasoning step and candidate action contracts for Stage 3.
  - [Episodic Consolidation Schema](./schema/consolidate.json) &mdash; Graph fusion receipt specifications for Stage 4.
  - [Closed-Loop Orchestration Schema](./schema/orchestrate.json) &mdash; Unified multi-stage execution and telemetry contracts.
- **Walkthrough Examples**:
  - [Unified Closed-Loop Cycle](./examples/closed-loop-cycle.md) &mdash; Complete multi-node incident response trace.
  - [Staged Invocation Runbook](./examples/staged-invocation.md) &mdash; Step-by-step intermediate execution.
  - [Fault Tolerance & Degraded Fallback](./examples/fallback-degraded.md) &mdash; Resilient handling of node timeouts, dual-memory fallback, and network partitions.
- **Evaluation & Verification**:
  - [Skill Calibration Guide](./eval.md) &mdash; Self-scoring evaluation instructions and scoring rubric links.

---

## 🛡️ Resilience & Degradation Protocols

Follow the tool's stage deadlines (see **Stage Deadlines**) and the retry rules under **Unified Closed-Loop Pipeline**, and degrade gracefully &mdash; never fabricate a stage result.

1. **Unconfigured Endpoint Guard**:
   - If any endpoint URL resolves to blank (`""`), the agent must halt and prompt the operator to run `sekha-cluster-tool env init` and populate `.env`. Never guess or hardcode addresses.

2. **Sensory Gating Fallback (Node 3 Unreachable)**:
   - If Node 3 encounters a connection failure or timeout, the agent preserves the raw sensory input by treating it as an unranked salient chunk with default unit salience (`1.0`), proceeding directly to Stage 2 (`recall`) without crashing.

3. **Scratchpad Timeout Guard & Trajectory Halting (Node 2 Unresponsive)**:
   - If Node 2 exceeds its Stage 3 deadline, or reports `over_budget`, the agent flags the degradation, halts the trajectory safely, and notifies the operator. It must never invent synthetic reasoning thoughts or uncommitted candidate actions.
   - **Enforceability across Invocation Modes**:
     - *Staged Path*: The agent controls Stage 4 directly and **MUST halt** before calling `sekha-cluster-tool consolidate`. Do not consolidate an incomplete or timed-out trajectory.
     - *Orchestrate Path*: Because all four stages run in-process within the tool backend, consolidation is not conditional on deliberation success and commits automatically. The agent cannot halt consolidation retroactively once invoked. If deliberation times out or fails during orchestration (exit code 2, `status` `partial`, `3_working_deliberate` failed), the agent must not re-run the cycle, and must report the polluted `session_id` and `trace_id` to the operator and state that the CLI exposes no rollback or delete subcommand for long-term graph mutations.

4. **Episodic Persistence Redundancy (Node 1 Offline)**:
   - If Stage 4 consolidation fails or Node 1 is offline, the **Dual-Write Contract guarantees zero amnesia**: the fact has already been safely persisted to Antigravity's auto-memory (resolved harness directory; never the user's working directory or repository).
   - The agent records the trace locally for deferred retry once Node 1 connectivity is restored. Primary task completion must not be blocked by background consolidation failures.
   - Exception: a `consolidate deadline of … exceeded` error is never retried automatically, because Node 1 may still complete that write. Suggest raising `CLUSTER_CONSOLIDATE_TIMEOUT_MS`, or one retry with `--consolidate-timeout` if the user agrees.

5. **Two-Tier Retrieval Degradation (Node 1 Recall Offline or Empty)**:
   - If Node 1 recall fails or returns empty results, immediately execute **Tier 2 Antigravity Memory Fallback** by reading Antigravity's auto-memory (resolved harness directory; never the user's working directory or repository).
   - Never fabricate or guess unretrieved parameters. If neither tier holds the data, decline honestly.

---

## 📐 Governance & Style Discipline

- **Pure British English (`en_GB`)**: All explanatory prose, reports, and documentation must adhere strictly to British English spelling (*initialise*, *serialise*, *optimise*, *neighbour*, *behaviour*, *prioritise*, *memorise*).
- **Frozen Contract Keys**: JSON keys emitted by `sekha-cluster-tool` mirror Go struct tags exactly (`salient_chunks`, `reduction_rate`, `sim_score`, `proposed_action`, `is_complete`, `loop_complete`, `trace_id`, `stages[]`, `latency_ms`). They must never be altered, re-cased, or anglicised.
- **Stage Deadlines**: Each stage's `duration_ms` must be read against its deadline (sensory 30 s, recall 1.5 s, deliberation widened from `CLUSTER_DELIBERATE_TIMEOUT_MS`, consolidation 120 s by default), alongside telemetry fields (`latency_ms`, `query_latency_ms`, `total_duration_ms`). A discrete `deliberate` call is budgeted with `--timeout 180s` (45 s for the preflight probe).
- **CLI Shell Safety**: All inline trace JSON payloads passed to `--trace` must be enclosed in single quotes (`'{"session_id":...}'`) to ensure clean execution under tool allowlists.
