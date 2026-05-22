# agicore-examples

**Agentic patterns done properly — deterministic, auditable, type-safe.**

A curated collection of real-world agentic systems implemented as single `.agi` files using the [Agicore DSL framework](https://github.com/Binary-Blender/agicore). Each example compiles to a complete, working Tauri desktop application.

---

## The problem with "agents" today

Most production "agents" are fragile chains of LLM calls glued together with Python:

- No type safety — a model returning `"null"` instead of `null` breaks the whole chain silently
- No retry logic — a timeout at step 3 means starting over from step 1
- No quality control — there is no way to know if the output is good until a human checks it
- No audit trail — you cannot replay what the model saw, what it decided, or why
- No governance — any LLM call can do anything; there are no enforced constraints

Each example in this repository replaces one of these fragile patterns with a clean declarative `.agi` file. The Agicore compiler generates deterministic Rust/Tauri code with typed inputs and outputs, structured retry policies, SPC-based quality control, SQLite-backed audit logs, and governance manifests — all as first-class DSL declarations.

---

## Repository layout

The repo is organized into four buckets by intent:

```
agicore-examples/
├── patterns/      # Focused single-pattern demos (one declaration cluster each)
├── accelerando/   # The Accelerando enterprise suite (18 vertical apps)
├── reference/     # Broader reference apps showing fuller Agicore use
└── showcase/      # Polished production-shape apps that don't track platform releases
```

### `patterns/` — Focused pattern demos

Each numbered example replaces a fragile agentic chain with a clean declarative `.agi` file. Examples 1 and 2 are fully implemented; 3–10 are syntactically valid stubs showing the shape of the solution.

| # | Folder | Name | Declarations showcased |
|---|--------|------|------------------------|
| 1 | `patterns/01-research-pipeline` | Research Pipeline | WORKFLOW, ACTION, PIPELINE |
| 2 | `patterns/02-support-router` | Support Router | ROUTER, SKILLDOC, TRIGGER, CHANNEL |
| 3 | `patterns/03-multi-model-council` | Multi-Model Council | AI_SERVICE, NBVE, ACTION |
| 4 | `patterns/04-automated-qa` | Automated QA | WORKFLOW, QC, TEST, ACTION |
| 5 | `patterns/05-data-extraction` | Data Extraction | ACTION, ENTITY, TYPE, PIPELINE |
| 6 | `patterns/06-document-rag` | Document RAG | SESSION, CHANNEL, PACKET, ACTION |
| 7 | `patterns/07-monitoring-loop` | Monitoring Loop | REASONER, TRIGGER, EVENT, LOG |
| 8 | `patterns/08-content-pipeline` | Content Pipeline | WORKFLOW, QC, COMPILER |
| 9 | `patterns/09-competitive-intel` | Competitive Intel | PIPELINE, SCORE, MODULE, ACTION |
| 10 | `patterns/10-moderation-loop` | Content Moderation | ROUTER, RULE, SCORE, SKILLDOC |

### `accelerando/` — Enterprise application suite

Eighteen `.agi` files demonstrating Agicore across a complete AI-native enterprise platform. Each app is a single source file that compiles to a running web service or desktop application. Every AI invocation happens at build time or in scheduled batch — nothing trusts an LLM at runtime.

**Enterprise Core (12):** `erp`, `billing`, `legal`, `lms`, `pi-coe`, `qms`, `oie`, `es`, `chatbot`, `eliza`, `config`, `interchange`

**EMR / Healthcare Stack (6):** `scheduling`, `clinical`, `radiology`, `pharmacy`, `population-health`, `patient-portal`

For the architectural narrative (entity catalogs, MODULE breakdowns, integration topology, the rule sets that make each domain deterministic), see [ACCELERANDO.md](https://github.com/Binary-Blender/agicore/blob/main/ACCELERANDO.md) in the main Agicore repo — that document is the architectural reference for what the suite proves about Agicore. The `.agi` sources you'd run are here.

### `reference/` — Broader reference apps

Twelve apps showing fuller Agicore use across mixed declaration clusters. Useful when learning how multiple Agicore features compose in a single application: `babyai-router`, `basketball-mmo`, `cognitive-infrastructure`, `content-pipeline`, `conversation-engine`, `creator-network`, `distributed-cognition`, `distributed-orchestration`, `home-academy`, `invoice-approval`, `organizational-intelligence`, `semantic-workflow`.

### `showcase/` — Polished production-shape apps

Apps that are real products, not stubs. They evolve on their own cadence rather than tracking platform releases.

| Folder | Name | What it is |
|---|---|---|
| `showcase/novasyn-mba` | NovaSyn MBA | Small-entrepreneur expert system app (1,227 lines, 41 declaration types — RULE, SKILL, WORKFLOW, EVENT). Second production reference application alongside NovaSyn Chat (which stays in the main agicore repo as the canary). |

### What lives where

| Repo | What's there | Cadence |
|---|---|---|
| [agicore](https://github.com/Binary-Blender/agicore) | Platform (compiler, core, DSL, prompts) + NovaSyn Chat as the canonical pinned-versions reference | Tied to platform releases |
| **agicore-examples** (this repo) | Patterns, the Accelerando suite, reference apps, showcase apps | Independent — pinned to specific Agicore versions per example |
| [novasyn](https://github.com/Binary-Blender/novasyn) | The AI-native spatial desktop / Phase 10 of the Agicore vision | Independent product

---

## How to use

### Prerequisites

- [Agicore CLI](https://github.com/Binary-Blender/agicore) installed
- Rust toolchain + Tauri prerequisites (`cargo`, `tauri-cli`)
- Node.js 20+

### Compile and run an example

```bash
# From the agicore repo root:
node core/compiler/dist/cli.js generate path/to/example.agi --output path/to/output/dir

# Then launch the generated Tauri app:
cd path/to/output/dir
npm install
cargo tauri dev
```

### What gets generated

Each `.agi` file is self-contained and compiles to a complete Tauri desktop application:

| Output | Description |
|--------|-------------|
| `migrations/` | SQLite schema — run these in order on first launch |
| `src-tauri/src/commands/` | Rust Tauri commands (CRUD, AI dispatch, routing) |
| `src/lib/` | TypeScript types, invoke wrappers, Zustand store |
| `src/components/` | React components (list views, forms, AI chat) |
| `src-tauri/tauri.conf.json` | Tauri configuration and ACL capabilities |

Any file with `// @agicore-protected` on line 1 is skipped on regen — your custom implementations survive.

---

## Framework

These examples are built on the [Agicore DSL framework](https://github.com/Binary-Blender/agicore).

> Agicore is a deterministic systems-authoring platform for AI-native organizations. The core principle: AI at build-time, determinism at runtime, DSL as constraint boundary.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
