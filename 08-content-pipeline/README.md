# 08 — Content Pipeline

**Brief → AI draft → critique loop → revision → SPC quality gate → publish.**

## The problem

AI content generation pipelines usually call the LLM once and ship whatever comes out. When quality is added, it's typically another LLM call asking "is this good?" — which has no memory of past outputs, no statistical baseline, and no mechanism to actually block a bad piece from being published.

## The agentic anti-pattern this replaces

```python
# Generate, maybe critique, hope for the best
draft = llm.complete(f"Write content about {brief}")
quality_check = llm.complete(f"Rate this content 1-10: {draft}")
if int(quality_check) >= 7:
    publish(draft)
```

## The Agicore approach

- `WORKFLOW content_workflow` — sequential: generate → critique → revise, with typed `ON_FAIL` handlers at each step
- `QC content_quality_gate` — Statistical Process Control gate: tracks quality scores across the last 20 outputs, blocks publishing if scores fall outside control limits (UCL/LCL). This is the same technique Toyota uses in manufacturing.
- `ACTION critique_draft` — structured critique with per-dimension scores (clarity, relevance, tone, keywords) and a `ready_to_publish` boolean
- `SESSION drafting_session` — tracks revision count and last QC score across the drafting session

## Key declarations

| Declaration | Why it's here |
|-------------|---------------|
| `WORKFLOW` | Orchestrates draft→critique→revise with explicit failure handling at each step |
| `QC` | SPC quality gate — uses statistical control limits, not vibes, to determine publish-readiness |
| `ACTION` | Typed AI dispatch — critique returns structured scores, not freeform text |
| `SESSION` | Maintains drafting context across multiple revision rounds |

## How to compile and run

```bash
cd path/to/agicore/core/compiler
node dist/cli.js generate path/to/agicore-examples/08-content-pipeline/content_pipeline.agi --output ~/my-content-app
cd ~/my-content-app && npm install && cargo tauri dev
```
