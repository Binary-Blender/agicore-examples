# Example 02 — Support Router

A customer support system that classifies incoming tickets by priority, routes them through a tiered AI specialist system (FAQ bot to specialized model to human escalation), enforces response governance via skill docs, and logs every routing decision to SQLite.

---

## What problem this solves

Customer support is a domain where AI routing can save hours of triage — but only if the routing is reliable and auditable. A ticket misclassified as low priority that turns out to involve fraud is a real business risk. This example implements a three-tier routing system with explicit governance: tier 1 (fast Haiku for FAQ), tier 2 (Sonnet for complex technical/policy), tier 3 (escalation for legal, fraud, or critical issues). Every routing decision is stored in SQLite as a `Response` entity with the tier recorded. The SKILLDOC governance layer enforces what operations each tier can perform.

---

## The agentic anti-pattern it replaces

The typical Python support router is a long `if/elif` chain that classifies based on keyword matching, then calls different prompts. Common failure modes:

- The keyword list misses edge cases — a billing fraud ticket gets routed to the FAQ tier
- The circuit breaker is a `try/except` with a sleep and hardcoded retry count
- There are no governance constraints on what the model can say or do — it can tell a customer to "just ignore that charge"
- The routing decision exists only in application logs, if those are even being written
- Adding a fourth tier means editing application code, redeploying, and hoping nothing breaks

Here, routing tiers are DSL declarations. Adding a tier 4 is a four-line edit to the ROUTER block followed by a compiler run. The SKILLDOC governance layer is a signed manifest, not a prompt comment the model can ignore. The CIRCUIT_BREAKER is a first-class field on tier 2, not a try/except block.

---

## Key Agicore declarations

**`TYPE TicketStatus`, `TYPE TicketPriority`**
Union type aliases. The compiler generates TypeScript union types and Rust string validation. A ticket cannot have a `status` of `"in_progress"` — it fails at the type boundary.

**`SKILLDOC support_guidelines`, `escalation_protocol`**
Two governed skill documents. `support_guidelines` is injected into tier-1 and tier-2 responses; `escalation_protocol` into tier-3. The GOVERNANCE block enforces: which operations are permitted (`EXECUTE_ONLY`), which are explicitly banned (`DISALLOW`), and what audit level to apply (`all_access` vs `all_actions`). The compiler generates a TypeScript registry with `isOperationPermitted()` and `buildSkillDocContext()` helpers.

**`ROUTER support_router_logic`**
Three tiers with explicit model assignments, strength tags, cost hints, and context limits. The tier-2 `CIRCUIT_BREAKER` declaration generates a Rust guard that falls back to tier 3 if tier 2 fails more than 30% of the time in a 120-second window. This is not a try/except — it is a stateful SPC-tracked failure detector.

**`ACTION classify_ticket`**
Returns structured JSON — priority, recommended tier, reasoning. Typed `OUTPUT` means the calling workflow receives a validated struct, not a raw string.

**`PACKET TicketEvent`, `CHANNEL ticket_channel`**
Typed SQLite-backed message bus. When a new ticket is created, the UI publishes a `TicketEvent` to `ticket_channel`. The TRIGGER listens on that channel and fires the `route_ticket` workflow. This decouples ticket creation from routing — the UI does not call the router directly.

**`TRIGGER on_new_ticket`**
Reactive binding: when a `TicketEvent` arrives on `ticket_channel`, fire `WORKFLOW route_ticket`. `IDEMPOTENT true` ensures the same ticket event cannot trigger two routing runs if the channel is processed twice.

**`WORKFLOW route_ticket`**
Two steps: classify, then respond. `ON_FAIL stop` on classify (a misclassified ticket is worse than a delayed one). `ON_FAIL retry` on respond (a transient model failure should retry before giving up).

---

## How to compile and run

```bash
# From the agicore repo root:
node core/compiler/dist/cli.js generate \
  path/to/agicore-examples/02-support-router/support_router.agi \
  --output path/to/output/support_router

cd path/to/output/support_router
npm install
cargo tauri dev
```

The app opens with the Ticket List. Create a new ticket with a subject and body. The classification action runs automatically, and the generated response appears in the Responses view with the routing tier recorded.

To run the generated Rust tests:

```bash
cd path/to/output/support_router/src-tauri
cargo test
```
