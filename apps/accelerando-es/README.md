# Accelerando Expert System

**The deterministic governance layer for the Accelerando platform.**

The OIE asks *"What is happening?"* — AI, probabilistic, retrospective.  
The Expert System asks *"What should happen right now?"* — rules, deterministic, instantaneous.

Zero LLMs at runtime. Every decision is auditable, reproducible, and explainable.  
Every rule firing has a traceable cause: which rule, which fact, which value, when.

---

## The Three-Tier Intelligence Stack

```
Accelerando ERP                 (business data — 32 entities, 32 actions)
        │
        │  every action → TelemetryPacket
        │
        ├──────────────────────────────────────────────────────┐
        ▼                                                      ▼
Expert System (deterministic)                    OIE (AI reasoning)
  Rules fire on current facts                    Patterns surface over time
  "Invoice overdue 60+ days → collections"       "AP team is 3× benchmark"
  "PO $47k → director approval required"         "Lead qualification gap widening"
  "Critical ticket unassigned 3h → page"         "Q3 cash flow risk emerging"
  "Stock below reorder point → trigger PO"       "Escalation spike correlates with deploy"
        │                                                      │
        │  DecisionPacket                                      │
        └──────────────► OIE telemetry ◄───────────────────────┘

OIE meta-insight: "CreditControl.block_new_orders fired 47 times this month — up 80%.
                   Collections process needs structural attention."
```

The Expert System's decisions **are telemetry** — the OIE reasons over rule-firing patterns just like it reasons over ERP actions. This creates a second-order intelligence layer: AI observing deterministic policy enforcement.

---

## Six Governance Modules

### CreditControl
AR enforcement without ambiguity.

```
CustomerBalance fact → 5 rules → CreditStatus state machine
  good → warning (utilization ≥ 80%) → blocked (overdue > 30d) → collections (60d) → legal (90d)
```

Rules: `credit_utilization_warning`, `block_new_orders`, `payment_history_risk`,  
`trigger_collections`, `escalate_to_legal`

### ApprovalMatrix
Spending authority policy as code. One place to read, one place to change.

```
SpendingRequest fact → 6 rules
  ≤ $1k + approved vendor    → auto_approved
  $1k–$10k                   → manager_approval_required
  $10k–$50k                  → director_approval_required
  > $50k                     → board_approval_required
  CAPEX > $5k                → capex_finance_review
  Unapproved vendor > $500   → unapproved_vendor_block
```

### SLAEnforcement
Prevent breaches. Act before, not after.

```
TicketAge fact → sla_risk_score (SCORE) → SLAStatus state machine → 5 rules
  Critical + 2h unassigned  → sla_breach_imminent_critical (PRIORITY 100)
  High + 8h unassigned      → sla_breach_imminent_high
  Enterprise + 1h unassigned → enterprise_unassigned_alert
  Normal > 20h              → sla_breach_approaching_normal
  3+ escalations            → chronic_escalation
```

### LeadScoring
CRM pipeline prioritization without an LLM.

```
LeadSignals fact → lead_score (SCORE with decay) → 5 rules + 5 patterns
  Score ≥ 80 → hot → assign_sales_rep
  Score ≥ 60 → qualified → qualify_lead
  Score ≥ 40 → warm → queue_for_nurture
  Score decays 5 per day (decay catches stale leads automatically)
  Patterns: demo_request +25, pricing_page +15, case_study +10, webinar +12, no_contact -15
```

### InventoryControl
Stock governance. Zero manual triggers needed.

```
StockLevel fact (with pre-computed booleans) → StockStatus state machine → 5 rules
  healthy → low (14d supply) → critical (7d supply) → stockout
  healthy → overstock (90d supply)
  is_below_reorder == true    → reorder_triggered
  is_stockout == true         → stockout_alert (PRIORITY 100)
  is_overstock == true        → overstock_detected
  Critical item + lead_time_risk → critical_item_reorder
```

### FinancialControls
AR aging cadence and cash flow runway.

```
ARAgingRecord fact → ARStatus state machine → 4 AR rules
  7–29d (no reminder sent) → payment_reminder
  30–59d (no final notice) → final_notice
  60d+                     → ar_collections
  Enterprise + 15d         → enterprise_overdue (elevated threshold)

CashPosition fact → CashFlowStatus state machine → 4 cash flow rules
  < 180d runway → cash_flow_watch
  < 90d runway  → cash_flow_warning (SEVERITY critical)
  < 30d runway  → cash_flow_critical (PRIORITY 100)
  High burn + < 120d runway → burn_rate_alert
```

---

## Why Deterministic Rules Beat LLMs for This

| Concern | LLM Approach | Expert System Approach |
|---|---|---|
| "Should this order be blocked?" | Ask the model, hope it says the right thing | `credit_utilization >= 100 AND days_overdue > 30 → FLAG credit_block` |
| Auditability | "The model decided." | Rule name, fact values, timestamp — admissible in court |
| Consistency | Varies by prompt, context, model version | Same input → same output, always |
| Speed | 200–2000ms per decision | Microseconds |
| Regulatory compliance | Impossible to prove | Trivial to prove |
| Cost | Per-token | Zero |

Use LLMs where uncertainty and judgment are genuinely needed (OIE insights, GenerateForecast, ExplainRuleFiring). Use rules where policy should be applied mechanically. The ES enforces the boundary.

---

## Architecture

```
accelerando_es.agi
│
├── ENTITY × 2                  → Audit trail (tenant_id on both)
│   RuleFiring                  → every rule firing, with fact_snapshot (SENSITIVE)
│   FactSnapshot                → entity state at evaluation time
│
├── PACKET × 2                  → Typed telemetry
│   DecisionPacket              → outbound to OIE (signed, admissible, 365d TTL)
│   FactPacket                  → inbound from ERP (entity snapshots)
│
├── CHANNEL × 2
│   erp_facts_in               → inbound from ERP
│   es_decisions_out           → outbound to OIE
│
├── MODULE × 6                  → Each module: FACT + RULE + STATE + SCORE + PATTERN
│   CreditControl              → 1 FACT, 5 RULES, 1 STATE (5 nodes)
│   ApprovalMatrix             → 1 FACT, 6 RULES
│   SLAEnforcement             → 1 FACT, 1 SCORE, 1 STATE, 5 RULES
│   LeadScoring                → 1 FACT, 1 SCORE, 5 PATTERNS, 5 RULES
│   InventoryControl           → 1 FACT, 5 RULES, 1 STATE (5 nodes)
│   FinancialControls          → 2 FACTs, 2 STATEs, 8 RULES
│
│   Total: 7 FACTs, 34 RULES, 2 SCOREs, 5 STATE MACHINES, 5 PATTERNS
│
├── ACTION × 6                  → Engine control + governance
│   FireRuleEngine             → evaluate all rules in a module, persist firings
│   RefreshFacts               → pull current ERP snapshots, compute derived fields
│   AcknowledgeAlert           → mark RuleFiring acknowledged
│   BulkAcknowledge            → batch acknowledge by module/severity
│   GenerateAuditReport        → AI-generated compliance report over firing history
│   ExplainRuleFiring          → AI plain-language explanation of a firing
│
├── VIEW × 4                    → Alert management console
│   AlertDashboard             → dashboard layout, all unacknowledged
│   UnacknowledgedAlerts       → kanban by severity
│   AlertHistory               → full audit table
│   FactBrowser                → inspect current fact values
│
├── PREFERENCE × 3              → rule_engine_interval, active_tenant, alert_retention
└── SEED × 5                    → demo firings matching OIE insight scenarios
```

---

## The Audit Story

"Why was Purchase Order #2891 blocked?"

```
SELECT * FROM rule_firings WHERE entity_id = 'po-2891';
→ {
    module_name:  "ApprovalMatrix",
    rule_name:    "require_director_approval",
    action_taken: "director_approval_required",
    severity:     "warning",
    fact_snapshot: { amount: 47500.00, requester_role: "manager", vendor_approved: true },
    fired_at:     "2026-05-20T10:30:00Z"
  }
```

The answer is complete, deterministic, and admissible. The rule text is in the DSL. The fact values are in the snapshot. The timestamp is exact.

No LLM involved. No prompt to reproduce. No "the model decided."

---

## Running the Full Accelerando Stack

```
accelerando_erp.agi   → web service (Axum + React + PostgreSQL + Docker)
accelerando_oie.agi   → desktop intelligence layer (Tauri + SQLite)
accelerando_es.agi    → desktop governance layer (Tauri + SQLite)

Data flow:
  ERP business actions → oie_telemetry_out → OIE.telemetry_ingress
  ERP entity snapshots → erp_facts_in → ES rule engine → es_decisions_out → OIE.telemetry_ingress

Both the OIE and ES feed the same intelligence pipeline.
The OIE sees ERP activity AND expert system decisions.
```

Together: one ERP + one intelligence layer + one governance layer.  
Three `.agi` files. One coherent AI-native enterprise platform.
