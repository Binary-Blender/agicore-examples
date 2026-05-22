# 07 — Monitoring Loop

**Periodic health checks → anomaly detection → event dispatch → reactive remediation. No polling loops in application code.**

## The problem

Monitoring systems built with agents typically run a loop that calls an LLM to "decide" whether something is wrong. The LLM has no memory of previous checks, no statistical baseline, and no typed contract for what an anomaly looks like. Every check is isolated.

## The agentic anti-pattern this replaces

```python
# Polling loop with no memory, no baseline, no typed events
while True:
    status = check_service(endpoint)
    if llm.complete(f"Is this healthy? {status}") == "no":
        send_alert(status)
    time.sleep(60)
```

## The Agicore approach

- `REASONER health_reasoner` — runs on a 60-second schedule, analyzes the rolling window of health checks, detects degradation patterns across time
- `EVENT ServiceDegraded` / `EVENT ServiceRecovered` — typed pub/sub events with idempotency guarantees and TTL
- `TRIGGER on_service_degraded` — reactive: when ServiceDegraded fires, run the alert workflow — no polling required
- `LOG` — every check and alert written to audit log with timestamps and severity

## Key declarations

| Declaration | Why it's here |
|-------------|---------------|
| `REASONER` | Scheduled periodic analysis — replaces polling loops with declared schedule + AI analysis |
| `EVENT` | Typed pub/sub — downstream systems subscribe to `ServiceDegraded`, not to a polling flag |
| `TRIGGER` | Reactive wiring — connects events to workflows without application glue code |
| `LOG` | Full audit trail — every check result and alert is logged deterministically |

## How to compile and run

```bash
cd path/to/agicore/core/compiler
node dist/cli.js generate path/to/agicore-examples/07-monitoring-loop/monitoring_loop.agi --output ~/my-monitoring-app
cd ~/my-monitoring-app && npm install && cargo tauri dev
```
