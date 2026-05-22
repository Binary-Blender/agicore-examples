# 09 — Competitive Intel

**Parallel competitor signal gathering → AI scoring → ranked strategic report.**

## The problem

Competitive intelligence agents typically run searches sequentially, dump everything into a single prompt, and ask the LLM to "summarize what matters." There's no scoring, no ranking, no way to tell which signals are high-confidence versus noise — and sequential execution is slow.

## The agentic anti-pattern this replaces

```python
# Sequential, unscored, no confidence tracking
for competitor in competitors:
    signals = search(competitor)
    report += llm.complete(f"What's notable about {competitor}? {signals}")
```

## The Agicore approach

- `PIPELINE intel_gather` — fan-out: gathers signals from all competitors in parallel, then scores each signal independently, then ranks by composite score before synthesis
- `SCORE signal_impact` — rates each signal on four weighted dimensions (market relevance, urgency, confidence, actionability). Only signals above threshold reach the final report.
- `MODULE competitive_analysis` — reusable expert bundle that activates when signal scores meet the threshold
- `ACTION synthesize_report` — synthesizes only the top-ranked signals into an executive briefing with explicit threat/opportunity/watch categorization

## Key declarations

| Declaration | Why it's here |
|-------------|---------------|
| `PIPELINE` | Parallel execution — all competitors analyzed simultaneously, not sequentially |
| `SCORE` | Weighted signal ranking — gates what reaches the final report by impact threshold |
| `MODULE` | Reusable expert bundle — encapsulates the analysis logic for reuse across apps |
| `ACTION` | Typed AI dispatch — synthesis receives structured ranked signals, not raw search dumps |

## How to compile and run

```bash
cd path/to/agicore/core/compiler
node dist/cli.js generate path/to/agicore-examples/09-competitive-intel/competitive_intel.agi --output ~/my-intel-app
cd ~/my-intel-app && npm install && cargo tauri dev
```
