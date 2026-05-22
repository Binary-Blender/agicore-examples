# Example 03 — Multi-Model Council

Broadcast a question to multiple AI models simultaneously, collect their independent answers, then synthesize the best combined response. Uses NBVE to promote a new synthesis model only after SPC quality gates pass — no blind upgrades.

---

## What problem this solves

A single model has blind spots. For high-stakes questions — architectural decisions, legal interpretations, medical information — you want diverse perspectives before committing to an answer. The council pattern solves this: send the same question to three or four models with no shared context, collect the independent answers, then synthesize. The synthesis step is itself an AI call, but it now has richer input than any single model could produce alone.

---

## The agentic anti-pattern it replaces

The typical Python council pattern is a `for model in models: responses.append(llm.call(model, prompt))` loop followed by a long concatenated context stuffed into a final synthesis call. Problems:

- If one model call times out, the list is silently shorter and the synthesis does not know
- The synthesis model is updated by changing a string in a config file — no quality gate, no rollback, no audit
- Response quality degrades gradually as models are swapped — no one notices until something goes obviously wrong

Here, each `CouncilResponse` is a typed entity in SQLite. The `NBVE` declaration gates synthesis model promotions through an SPC quality check: the shadow model must demonstrate >= 85% accuracy and >= 90% stability over 50 samples before it can replace the production model. Failed promotions fall back automatically.

---

## Key Agicore declarations

**`AI_SERVICE`** — four models across three providers. Genuine diversity, not just two versions of the same family.

**`NBVE council_nbve`** — No-Blind-Version-Elevation. Declares the production synthesis model, the shadow candidate, quality thresholds, and that promotion is `manual` (requires human sign-off even after SPC gates pass). This is the declaration that prevents silent quality regressions.

**`ACTION broadcast_question`** — stub for the fan-out action. Full implementation fans to all declared models concurrently, stores each as a `CouncilResponse` entity, and returns the collected responses as JSON.

**`ACTION synthesize_council`** — takes all council responses as JSON, reasons about consensus and disagreement, returns a synthesis with a confidence score.

---

## How to compile and run

```bash
node core/compiler/dist/cli.js generate \
  path/to/agicore-examples/03-multi-model-council/multi_model_council.agi \
  --output path/to/output/multi_model_council

cd path/to/output/multi_model_council
npm install
cargo tauri dev
```

Note: `broadcast_question` uses `IMPL` for the custom fan-out logic. After generating, implement the Rust stub in `src-tauri/src/commands/broadcast_question.rs`.
