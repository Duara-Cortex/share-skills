# Cluster Memory Skill

A declarative skill that lets an AI agent externalise memory and reasoning onto the **Sekha Tri-Node Edge Cognitive Cluster**, instead of holding an entire noisy stream and every recalled fact in its context window.

The skill does not talk to the nodes directly. It drives the compiled **[`sekha-cluster-tool`](https://github.com/Duara-Cortex/sekha-cluster-tool)** CLI (`>= v1.0.1`), which coordinates the cluster deterministically. A compiled binary keeps behaviour stable and prevents runtime drift across harnesses.

> This is the **claude** variant. An antigravity variant lives alongside it under `skills/antigravity/memory/cluster-memory/`.

---

## What it does

Cognition is routed through four discrete stages, each served by a dedicated node and exposed as a CLI subcommand that prints strict JSON to `stdout`:

| Stage | Subcommand | Cognitive layer | Purpose |
| :--- | :--- | :--- | :--- |
| 1. Sensory gating | `filter` | Sensory Attention Gate (`:8081`) | Reduce a noisy stream to salient signal |
| 2. Associative recall | `recall` | Knowledge Graph Store (`:8084`) | Ground the signal in long-term memory |
| 3. Working deliberation | `deliberate` | Working Scratchpad (`:8083`) | Reason a multi-step action |
| 4. Episodic consolidation | `consolidate` | Knowledge Graph Store (`:8084`) | Commit the episode so future recall improves |
| Full closed loop | `orchestrate` | All three layers | Run all four stages in one traced turn |

## When to use it

- A raw, noisy, or high-volume stream (syslog, telemetry, sensor text) must be reduced before reasoning.
- A task needs grounding facts or related entities from long-term memory before acting.
- A multi-step decision benefits from an explicit, inspectable reasoning trajectory.
- A completed episode should be persisted to long-term memory.

Skip it when the task is self-contained and needs no external memory.

## Prerequisites

1. **`sekha-cluster-tool >= v1.0.1`** on `PATH`. Verify with `sekha-cluster-tool --version`.
2. **Configured endpoints.** All node addresses default to blank and must be supplied — there are no baked-in IPs.

## Configuration

Endpoints are resolved with 12-factor precedence: **CLI flags → OS environment variables → `.env` file → compile-time defaults → blank.**

```bash
# 1. Write a blank starter .env in the working directory
sekha-cluster-tool env init

# 2. Populate the three endpoints (and optional tuning params) in .env:
#    CLUSTER_SENSORY_URL, CLUSTER_WORKING_URL, CLUSTER_KNOWLEDGE_URL

# 3. Confirm the resolved configuration
sekha-cluster-tool env show

# 4. Check reachability and per-node latency before a live run
sekha-cluster-tool status
```

Never hard-code cluster IP addresses — in the skill, in examples, or in agent output. Reference the tool and its `.env` configuration only.

## Usage

**Preferred — one traced turn:**
```bash
sekha-cluster-tool orchestrate \
  --input "<raw stream>" \
  --directive "<goal>" \
  --session-id "<id>" \
  --sync
```

**Staged — inspect or intervene between stages:**
```bash
sekha-cluster-tool filter      --text "<raw stream>" --directive "<what to attend to>" --threshold 0.45
sekha-cluster-tool recall      --query "<salient concept>" --top-k 5
sekha-cluster-tool deliberate  --task "<objective>" --input "<observation>" --context "<grounding fact>"
sekha-cluster-tool consolidate --goal "<objective>" --outcome "success" --session-id "<id>" --sync
```

Every request carries an `X-Trace-ID` so operations can be correlated across cluster logs. Diagnostic step logs go to `stderr` under `--verbose`; only `stdout` carries the JSON contract.

## Graceful degradation

The skill never fabricates a stage result. If the tool is missing or endpoints are blank, it stops and asks you to configure them. If a node is unreachable it degrades transparently: sensory/recall failures proceed with reduced confidence and a flagged stage, a scratchpad (Stage 3) failure halts the loop, and a consolidation failure still returns the reasoned action while noting the episode was not persisted. See [`examples/fallback-degraded.md`](examples/fallback-degraded.md).

## Package layout

```
cluster-memory/
├── SKILL.md          # Model-facing instructions (the actual skill)
├── README.md         # This file (human-facing overview)
├── eval.md           # Calibration guide + pointers to the central eval assets
├── schema/           # JSON request/response contracts per subcommand
│   ├── filter.json
│   ├── recall.json
│   ├── deliberate.json
│   ├── consolidate.json
│   └── orchestrate.json
└── examples/         # Worked multi-turn walkthroughs
    ├── closed-loop-cycle.md
    ├── staged-invocation.md
    └── fallback-degraded.md
```

The `schema/` files document the canonical payload contracts (mirroring the tool's response models). They are a reference; the keys the running binary actually emits are authoritative for any given run.

## Evaluation

The skill ships with a self-scoring eval. From the `share-skills` repo root:

```
/eval claude/cluster-memory
```

This loads `SKILL.md` as the system prompt, replays the multi-turn fixtures in [`evals/fixtures/cluster-memory.json`](../../../../evals/fixtures/cluster-memory.json) (with simulated tool output — no live cluster required), and grades the transcript against [`evals/rubrics/cluster-memory.md`](../../../../evals/rubrics/cluster-memory.md) using the calibration guide in [`eval.md`](eval.md). See [`eval.md`](eval.md) for details.

## Conventions

- **Decoupled:** zero hardcoded IPs; all endpoints come from `sekha-cluster-tool` `.env` configuration.
- **Licence:** Apache 2.0 (see the repository `LICENSE`).
