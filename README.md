# Share Skills

A curated, production-grade repository of modular skills, commands, cognitive workflows, and self-scoring evaluation harnesses crafted by the **Duara-Cortex** team. Designed to inject robust domain knowledge, behavioural rules, and execution capabilities directly into AI coding agents and autonomous cognitive systems.

---

## 🏛️ Purpose & Philosophy

In modern agentic development, treating Markdown context as infrastructure allows teams to build predictable, deterministic, and high-performing AI workflows. Rather than relying on fragile ad-hoc prompting, each skill package in this repository serves as an immutable, version-controlled runbook and instruction set.

### Core Tenets

1. **Context as Infrastructure**: Procedural instructions, domain rules, and tool contracts are treated with the same engineering rigor as application source code.
2. **Progressive Disclosure**: To preserve the agent's context window, instruction sets are layered. The agent ingests the high-level router (`SKILL.md`) first and resolves payload schemas (`schema/`), operational traces (`examples/`), and evaluation guides (`eval.md`) dynamically on-demand.
3. **Decoupled Tooling & Determinism**: For complex, stateful, or hardware-level operations (such as distributed edge clustering), skills interface with compiled CLI binaries (e.g. [`sekha-cluster-tool`](https://github.com/Duara-Cortex/sekha-cluster-tool)). A compiled harness eliminates runtime prompt mutation and guarantees sub-second latency budgets.

---

## 📂 Repository Structure

The repository is organised into three primary components: commands, central evaluation assets, and platform-targeted skill packages.

```text
share-skills/
├── commands/
│   └── eval.md                     # Generic /eval test runner command protocol
├── evals/
│   ├── fixtures/                   # Central multi-turn synthetic test inputs and stream fixtures
│   │   └── cluster-memory.json     # Fixtures for the Tri-Node Cluster Memory suite
│   └── rubrics/                    # Calibrated objective PASS (1) / FAIL (0) evaluation rubrics
│       └── cluster-memory.md       # 7-category scoring rubric for cluster memory
├── skills/
│   ├── antigravity/                # Skills crafted specifically for Google Antigravity (AGY)
│   │   └── memory/
│   │       └── cluster-memory/     # Tri-Node Edge Cognitive Cluster skill package
│   │           ├── README.md       # Human-readable package documentation
│   │           ├── SKILL.md        # Agent system prompt, CLI contracts & fallback rules
│   │           ├── eval.md         # Co-located calibration guide referencing central evals
│   │           ├── schema/         # JSON schemas for filter, recall, deliberate, consolidate, orchestrate
│   │           └── examples/       # Multi-turn traces for closed-loop, staged, and fallback modes
│   ├── claude/                     # Skills structured for Claude Code CLI and Anthropic agent loops
│   ├── custom-harness/             # Skills designed for bespoke agent orchestrators and runtimes
│   └── slm/                        # Compact skills optimised for edge Small Language Models
├── LICENSE                         # Apache 2.0 Licence
└── README.md                       # Repository overview and architecture guide
```

---

## 🎯 Target Platforms

Skills in this repository are categorised by target agent platform to leverage platform-specific tool interfaces, context management, and activation mechanics:

| Platform | Directory | Best For | Description |
| :--- | :--- | :--- | :--- |
| **Antigravity** | `skills/antigravity/` | Google Antigravity IDE / CLI | Formatted for AGY progressive disclosure, system rules, and workspace integration. |
| **Claude** | `skills/claude/` | Claude Code CLI | Tailored for Anthropic agent loops, Bash tools, and slash-command structures. |
| **Custom Harness** | `skills/custom-harness/` | Bespoke Orchestrators | Generic declarative specifications for autonomous loops and multi-agent systems. |
| **SLM** | `skills/slm/` | Small Language Models | Token-dense, low-complexity instruction sets optimised for local edge inference (4B–14B models). |

---

## 🧠 Cognitive Domains & Functional Categories

Skills are grouped into functional domain subdirectories within each platform:

- **`memory/`**: Advanced multi-tier cognitive architectures, sensory attention gating, associative knowledge graph retrieval, working memory scratchpad deliberation, and episodic trace consolidation (e.g. [`cluster-memory`](skills/antigravity/memory/cluster-memory/)).
- **`automation/`**: Multi-stage workflow pipelines, message bus coordination, and chain-of-responsibility handlers.
- **`coding/`**: Pair-programming workflows, conventional git commit generators, automated multi-agent code reviews, and refactoring linters.

---

## 🧪 Self-Scoring Evaluation Harness (`/eval`)

To guarantee prompt quality, prevent behavioral regression, and verify that agents strictly adhere to tool contracts, this repository features an integrated self-scoring evaluation framework:

1. **Centralised Assets**:
   - **Fixtures** (`evals/fixtures/*.json`): Multi-turn test scenarios simulating user prompts, sensor streams, and simulated tool outputs on `stdout`.
   - **Rubrics** (`evals/rubrics/*.md`): Objective, binary **PASS (1)** / **FAIL (0)** scoring criteria with explicit calibration boundaries.
2. **Co-located Mappings**:
   - Each skill directory contains an `eval.md` file declaring its associated central fixtures and rubric in its YAML frontmatter:
     ```yaml
     ---
     fixtures: ../../../../evals/fixtures/cluster-memory.json
     rubric: ../../../../evals/rubrics/cluster-memory.md
     ---
     ```
3. **Execution Command**:
   - Run self-scoring evaluations programmatically or interactively via the `/eval` runner defined in [`commands/eval.md`](commands/eval.md):
     ```bash
     /eval antigravity/cluster-memory
     ```
   - An evaluator agent parses the transcript turn-by-turn, scores all applicable criteria, and compiles an audit report with an aggregate rating (**Excellent** $\ge 90\%$, **Good** $75-89\%$, **Needs Work** $60-74\%$, **Poor** $<60\%$).

---

## 🛠️ Environmental Decoupling

All skills strictly enforce 12-factor environmental decoupling:
- **Zero Hardcoded IPs**: Private infrastructure addresses and hardware endpoints are never hardcoded in markdown files or schemas.
- **`.env` Precedence**: Endpoint configuration is dynamically resolved through local `.env` files, operating system environment variables, or CLI flags.
- **Starter Templates**: Companion tools provide interactive initialisation (e.g. `sekha-cluster-tool env init` or `make env`).

---

## 📄 Licence

Apache 2.0. Authored and maintained by the **Duara-Cortex** team.
