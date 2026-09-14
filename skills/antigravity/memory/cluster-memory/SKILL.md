---
name: cluster-memory
description: >-
  Coordinates the Tri-Node Edge Cognitive Cluster across sensory attention gating (Node 3), associative knowledge recall (Node 1), working scratchpad deliberation (Node 2), and episodic memory consolidation (Node 1). Use when processing edge telemetry streams, retrieving grounded facts, executing deliberate multi-step reasoning, or persisting episodic memory traces via sekha-cluster-tool.
---

# Cluster Memory Skill

The **Cluster Memory Skill** governs distributed cognitive workflows across a dedicated tri-node edge computing cluster. It interfaces exclusively through the compiled Go CLI harness [`sekha-cluster-tool`](https://github.com/Duara-Cortex/sekha-cluster-tool) to eliminate behavioural drift and ensure strict contract enforcement.

---

## 🏛️ Cognitive Topology & Architecture

The cluster partitions cognitive responsibilities across three discrete hardware tiers:

```
[Raw Telemetry / Sensory Stream]
               │
               ▼
    Stage 1: Sensory Attention Gate (Node 3 &bull; :8081)
    Lightweight CPU entropy and density salience gating (<1ms)
               │
               ▼
    Stage 2: Long-Term Knowledge Graph Store (Node 1 &bull; :8084)
    Associative recall: top-k vector similarity + recency decay
               │
               ▼
    Stage 3: Working Memory Scratchpad (Node 2 &bull; :8083)
    Multi-step deliberation with local SLM inference
               │
               ▼
    Stage 4: Memory Consolidation & Decay (Node 1 &bull; :8084)
    Episodic trace commitment for graph fusion and Hebbian decay
```

### Physical Node Allocation
- **Node 3** (`4GB RAM`): Hosts the in-memory ring buffer daemon and sensory attention gate on port `:8081`.
- **Node 2** (`16GB RAM`): Hosts the working memory scratchpad and local SLM inference engine on port `:8083`.
- **Node 1** (`8GB RAM`): Hosts the persistent knowledge graph store, vector indices, and consolidation daemon on port `:8084`.

---

## 📋 Prerequisites & Tooling

1. **Compiled Binary**: `sekha-cluster-tool` ($\ge \text{v1.0.1}$) must be installed and executable in the system `PATH`.
   - Verify installation:
     ```bash
     sekha-cluster-tool --help
     ```
   - Release installer:
     ```bash
     curl -fsSL https://raw.githubusercontent.com/Duara-Cortex/sekha-cluster-tool/main/scripts/install.sh | bash
     ```

2. **Decoupled Configuration**:
   - Zero IP addresses are hardcoded in the codebase or skill files.
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

# Timeout Budgets (milliseconds)
CLUSTER_DEFAULT_TIMEOUT_MS=1500
CLUSTER_DELIBERATE_TIMEOUT_MS=8000

# Attention & Recall Hyperparameters
CLUSTER_SALIENCE_THRESHOLD=0.45
CLUSTER_RECALL_TOP_K=5
```

Configuration resolution adheres strictly to 12-factor standard precedence:
1. Explicit CLI Flags (`--sensory-url`, `--working-url`, `--knowledge-url`)
2. Operating System Environment Variables
3. Local `.env` file (loaded from working directory, `--env-file`, or `~/.config/sekha-cluster-tool/.env`)
4. Compile-time injected builder defaults
5. Blank fallback (`""`)

---

## ⚙️ Cognitive Invocations & Commands

All CLI commands emit structured, un-padded JSON on `stdout`. Operational logs and diagnostics are directed to `stderr`.

### 1. Cluster Health & Latency Probe (`status`)
Probe reachability and measure roundtrip network latency across all three endpoints:
```bash
sekha-cluster-tool status
```

### 2. Stage 1: Sensory Attention Gating (`filter`)
Filter high-frequency raw telemetry, syslog streams, or text payloads to discard routine noise and isolate salient chunks:
```bash
sekha-cluster-tool filter \
  --text "<raw telemetry stream>" \
  --directive "Identify operational anomalies" \
  --threshold 0.45
```
*Schema details: [schema/filter.json](./schema/filter.json)*

### 3. Stage 2: Associative Knowledge Recall (`recall`)
Query the long-term knowledge graph on Node 1 for grounded facts, active policies, runbooks, and relational edges:
```bash
sekha-cluster-tool recall \
  --query "<salient concept or entity>" \
  --top-k 5
```
*Schema details: [schema/recall.json](./schema/recall.json)*

### 4. Stage 3: Working Memory Deliberation (`deliberate`)
Dispatch the task objective, salient sensory chunk, and distilled grounding facts to the Node 2 scratchpad for multi-step reasoning:
```bash
sekha-cluster-tool deliberate \
  --task "<task objective>" \
  --input "<sensory observation>" \
  --context "<retrieved facts and policy guidelines>"
```
*Schema details: [schema/deliberate.json](./schema/deliberate.json)*

### 5. Stage 4: Episodic Consolidation (`consolidate`)
Commit completed reasoning trajectories, actions, and outcomes to Node 1 for graph fusion, Hebbian reinforcement, and background decay:
```bash
sekha-cluster-tool consolidate \
  --goal "<task objective>" \
  --outcome "success" \
  --session-id "<session id>" \
  --sync
```
*Schema details: [schema/consolidate.json](./schema/consolidate.json)*

### 6. Unified Closed-Loop Cycle (`orchestrate`)
Execute the full 4-stage cognitive cycle in a single coordinated invocation:
```bash
sekha-cluster-tool orchestrate \
  --input "<raw sensory stream>" \
  --directive "<attention directive>" \
  --trace-id "<trace id>" \
  --sync
```
*Schema details: [schema/orchestrate.json](./schema/orchestrate.json)*

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
  - [Fault Tolerance & Degraded Fallback](./examples/fallback-degraded.md) &mdash; Resilient handling of node timeouts and partitions.
- **Evaluation & Verification**:
  - [Skill Calibration Guide](./eval.md) &mdash; Self-scoring evaluation instructions and scoring rubric links.

---

## 🛡️ Resilience & Degradation Protocols

1. **Unconfigured Endpoint Guard**:
   - If any endpoint URL resolves to blank (`""`), the agent must halt and prompt the operator to run `sekha-cluster-tool env init`. Never guess or hardcode addresses.

2. **Sensory Gating Fallback (Node 3 Unreachable)**:
   - If Node 3 encounters a connection failure or timeout, the agent preserves the raw sensory input by treating it as an unranked salient chunk with default unit salience (`1.0`), proceeding directly to Stage 2 without aborting.

3. **Scratchpad Timeout Guard (Node 2 Unresponsive)**:
   - If Node 2 exceeds `CLUSTER_DELIBERATE_TIMEOUT_MS`, the agent flags the degradation, halts the trajectory safely, and notifies the operator. It must never invent synthetic reasoning thoughts or uncommitted candidate actions.

4. **Episodic Persistence Failure (Node 1 Offline)**:
   - If Stage 4 consolidation fails, the agent records the episode locally for deferred retry. Primary task completion must not be blocked by background consolidation failures.

---

## 📐 Governance & Style Discipline

- **Pure British English (`en_GB`)**: All explanatory prose and user-facing reports must adhere strictly to British English spelling (*initialise*, *serialise*, *optimise*, *neighbour*, *behaviour*, *prioritise*).
- **Frozen Contract Keys**: JSON keys emitted by `sekha-cluster-tool` mirror Go struct tags exactly (`salient_chunks`, `reduction_rate`, `sim_score`, `proposed_action`, `is_complete`, `trace_id`, `latency_ms`). They must never be altered, re-cased, or anglicised.
- **Latency Budgets**: Sub-second roundtrips ($< 1000\text{ ms}$ per hop) must be verified against telemetry fields (`latency_ms`, `query_latency_ms`, `total_duration_ms`).
