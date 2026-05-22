# 10 — Content Moderation Loop

**Content in → deterministic pre-filter → AI classification → score-based routing → governed decision → full audit trail.**

## The problem

Content moderation agents typically make a single LLM call that both classifies the content and decides the action. There's no separation between detection and decision, no deterministic pre-filter for obvious cases, no appeal path, and no audit trail that a compliance team can actually read.

## The agentic anti-pattern this replaces

```python
# One call does everything — classify, decide, act, no audit
decision = llm.complete(f"Should this content be removed? Answer yes/no: {content}")
if decision == "yes":
    remove(content)
```

## The Agicore approach

- `RULE` declarations handle the obvious cases deterministically — known spam patterns, repeat offenders, legal keywords. No AI needed for these.
- `SCORE violation_confidence` — rates each moderation decision on four dimensions (classification confidence, severity, context clarity, repeat signal). The score determines which tier handles it.
- `ROUTER moderation_router` — three tiers: auto-moderation (fast/cheap), human-assisted (capable model), legal review (escalation). Circuit breakers prevent runaway costs.
- `SKILLDOC moderation_guidelines` + `SKILLDOC legal_escalation_protocol` — governance layer: what operations are permitted at each tier, which clearance levels are required, full audit of every decision.
- `LOG` — every classification, routing decision, and outcome written to an immutable audit log.

## Key declarations

| Declaration | Why it's here |
|-------------|---------------|
| `RULE` | Deterministic pre-filter — obvious violations handled without AI cost |
| `SCORE` | Routes to the appropriate tier by composite confidence score |
| `ROUTER` | Tiered routing with circuit breakers — cost-controlled escalation path |
| `SKILLDOC` | Governance layer — permitted operations, required clearance, audit level per tier |
| `LOG` | Immutable audit trail — compliance requirement, not an afterthought |

## How to compile and run

```bash
cd path/to/agicore/core/compiler
node dist/cli.js generate path/to/agicore-examples/10-moderation-loop/moderation_loop.agi --output ~/my-moderation-app
cd ~/my-moderation-app && npm install && cargo tauri dev
```
