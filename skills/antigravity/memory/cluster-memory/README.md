# Cluster Memory Skill for Antigravity

This folder contains the **Cluster Memory Skill** for Antigravity. It equips the agent with procedural instructions, JSON contract definitions, and execution runbooks to coordinate a distributed **Tri-Node Edge Cognitive Cluster** via the compiled Go CLI harness [`sekha-cluster-tool`](https://github.com/Duara-Cortex/sekha-cluster-tool).

---

## 🏛️ Design Philosophies

This skill adheres to core agentic and architectural design principles:

### 1. Compiled Determinism & Drift Elimination
All cognitive interactions with physical cluster nodes are routed through the compiled binary `sekha-cluster-tool` ($\ge \text{v1.0.1}$). A compiled harness prevents runtime prompt mutation, eliminates behavioural drift across sessions, and enforces strict sub-second timeout budgets (<1s per hop).

### 2. Decoupled 12-Factor Configuration
Zero cluster IP addresses are hardcoded in prompts, schemas, or markdown documents. Endpoint addresses default to blank (`""`) and are dynamically resolved via `.env` configuration, operating system environment variables, or CLI flags.

### 3. Progressive Disclosure
To preserve the agent's context window, instructions are split modularly:
- The agent reads `SKILL.md` for core cognitive routing and subcommand invocation syntax.
- Payload specifications are resolved on-demand from `schema/`.
- Session traces and failure recovery procedures are loaded from `examples/`.
- Evaluation calibration is decoupled into `eval.md` pointing to central fixtures and rubrics.

---

## 📂 Folder Structure

```text
cluster-memory/
├── README.md                 # This human-readable guide
├── SKILL.md                  # Main cognitive router, CLI contracts, and fallback protocols
├── eval.md                   # Evaluation calibration guide referencing central fixtures and rubrics
├── schema/                   # JSON schemas defining input/output contracts
│   ├── filter.json           # Stage 1 Sensory Attention Gate contract (:8081)
│   ├── recall.json           # Stage 2 Long-Term Knowledge Recall contract (:8084)
│   ├── deliberate.json       # Stage 3 Working Scratchpad Deliberation contract (:8083)
│   ├── consolidate.json      # Stage 4 Episodic Memory Consolidation contract (:8084)
│   └── orchestrate.json      # Unified 4-Stage Closed-Loop Orchestration contract
└── examples/                 # Empirical session traces and walkthroughs
    ├── closed-loop-cycle.md  # End-to-end incident mitigation trace
    ├── staged-invocation.md  # Step-by-step intermediate execution runbook
    └── fallback-degraded.md  # Fault tolerance, timeout, and degradation protocols
```

---

## 🔄 Cognitive Execution Protocol

The cluster executes a 4-stage closed-loop cognitive cycle across three distributed hardware nodes:

```
[Raw Sensory Input]
         │
         ▼
 Stage 1: Sensory Attention Gate (Node 3 • :8081)
 Discard routine noise; isolate salient chunks
 (CLI: `sekha-cluster-tool filter --text ...`)
         │
         ▼
 Stage 2: Long-Term Knowledge Graph (Node 1 • :8084)
 Retrieve associative runbooks, policies, and relational edges
 (CLI: `sekha-cluster-tool recall --query ...`)
         │
         ▼
 Stage 3: Working Memory Scratchpad (Node 2 • :8083)
 Deliberate multi-step reasoning with local SLM inference
 (CLI: `sekha-cluster-tool deliberate --task ... --input ... --context ...`)
         │
         ▼
 Stage 4: Episodic Consolidation (Node 1 • :8084)
 Commit trajectory, reinforce graph edges, and trigger Hebbian decay
 (CLI: `sekha-cluster-tool consolidate --goal ... --outcome ... --sync`)
```

For unified operations, `sekha-cluster-tool orchestrate --input "<stream>"` coordinates all four stages within a single invocation, injecting a distributed `X-Trace-ID` across every hop.

---

## 🧪 Running Self-Scoring Evaluations

To verify that the agent conforms to these invocation contracts and fallback protocols, execute the evaluation command:

```bash
/eval antigravity/cluster-memory
```

This runs the central multi-turn fixtures ([evals/fixtures/cluster-memory.json](../../../../evals/fixtures/cluster-memory.json)) and scores generated outputs objectively against the calibrated rubric ([evals/rubrics/cluster-memory.md](../../../../evals/rubrics/cluster-memory.md)).

---

## 🔗 Related Resources

- [SKILL.md](SKILL.md) &mdash; Full skill instructions and operational guidelines.
- [Calibration Guide](eval.md) &mdash; Evaluation scoring criteria and calibration boundaries.
- [Upstream Tool Repository](https://github.com/Duara-Cortex/sekha-cluster-tool) &mdash; Source code, build targets, and release packages for `sekha-cluster-tool`.
