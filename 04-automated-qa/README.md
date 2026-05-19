# Example 04 — Automated QA

An AI-powered quality assurance system that evaluates generated content against defined criteria, uses SPC-tracked quality control to detect systemic degradation, and routes low-quality outputs back for revision automatically.

---

## What problem this solves

When you use AI to generate content at scale, quality degrades silently. A model that passed quality checks last month may start producing subpar output due to prompt drift, model updates, or shifting input distributions. This example adds a QC gate to every content generation run — each output is scored against explicit criteria, the score is tracked by an SPC engine, and the system alerts when pass rates fall below statistical thresholds.

---

## The agentic anti-pattern it replaces

Most "automated QA" pipelines are a second LLM call that says "is this good? yes/no" with no memory across runs. There is no baseline, no trend tracking, no alert when quality starts drifting. The `QC` declaration generates an SPC engine backed by SQLite that tracks pass/fail rates across runs, distinguishes young (learning) from mature (stable) samples, and raises a flag when defect rates exceed control limits.

---

## Key Agicore declarations

**`QC content_quality`** — SPC-based quality control. `YOUNG_THRESHOLD 30` means the system is in a learning phase for the first 30 samples. `MATURE_PASS_RATE 0.92` means the mature system must pass 92% of samples or the QC declaration triggers an alert. `MATURING_SAMPLE 0.40` means 40% of maturing-phase outputs are checked (sampling efficiency).

**`WORKFLOW qa_workflow`** — three steps: generate, evaluate, revise. `ON_FAIL stop` on generate (no content = nothing to evaluate). `ON_FAIL stop` on evaluate (an unscored output cannot enter the pipeline). `ON_FAIL skip` on revise (if revision fails, keep the original with its QA feedback attached rather than losing the output entirely).

**`ACTION evaluate_content`** — returns structured JSON: `passed` (bool), `score` (float), `feedback` (string). The typed OUTPUT means the workflow step receives a validated struct.

---

## How to compile and run

```bash
node core/compiler/dist/cli.js generate \
  path/to/agicore-examples/04-automated-qa/automated_qa.agi \
  --output path/to/output/automated_qa

cd path/to/output/automated_qa
npm install
cargo tauri dev
```
