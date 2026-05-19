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

## Examples

| # | Folder | Name | Declarations showcased |
|---|--------|------|------------------------|
| 1 | `01-research-pipeline` | Research Pipeline | WORKFLOW, ACTION, PIPELINE |
| 2 | `02-support-router` | Support Router | ROUTER, SKILLDOC, TRIGGER, CHANNEL |
| 3 | `03-multi-model-council` | Multi-Model Council | AI_SERVICE, NBVE, ACTION |
| 4 | `04-automated-qa` | Automated QA | WORKFLOW, QC, TEST, ACTION |
| 5 | `05-data-extraction` | Data Extraction | ACTION, ENTITY, TYPE, PIPELINE |
| 6 | `06-document-rag` | Document RAG | SESSION, CHANNEL, PACKET, ACTION |
| 7 | `07-monitoring-loop` | Monitoring Loop | REASONER, TRIGGER, EVENT, LOG |
| 8 | `08-content-pipeline` | Content Pipeline | WORKFLOW, QC, COMPILER |
| 9 | `09-competitive-intel` | Competitive Intel | PIPELINE, SCORE, MODULE, ACTION |
| 10 | `10-moderation-loop` | Content Moderation | ROUTER, RULE, SCORE, SKILLDOC |

Examples 1 and 2 are fully implemented. Examples 3–10 are syntactically valid stubs showing the shape of the solution.

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
