# Accelerando Eliza — Operator Interface

**Natural language front-end for deterministic ERP workflow execution.**

> "Are you AI?"  
> No.  
> "Are you human?"  
> No.  
> "What are you?"  
> A very articulate button.

Eliza is a conversational macro executor. Operators speak; Eliza matches the intent to a compiled workflow and executes it. AI ran once at build time to generate the PATTERN declarations from standard operating procedures. Zero LLM at runtime. Same sentence, same workflow, every time.

This is not a chatbot. It's not a GUI. It's a third thing: deterministic workflow execution with a conversational interface.

---

## What This Solves

Every ERP system ships with a 400-page training manual because the UI is a labyrinth. New operators need weeks of training to navigate menus, find the right form, fill the right fields in the right order, click the right button sequence.

With Eliza that entire burden disappears. The operator says what they want in plain language. Eliza executes the exact correct workflow. The training manual becomes the chatbot — AI reads it at build time and compiles it into PATTERN declarations.

```
Without Eliza:
  Navigate to Sales → Opportunities → [find record] → Actions → Convert
  → fill Invoice form → add line items → Save → navigate to Projects
  → Create → link to Opportunity → assign team → Save

With Eliza:
  "Close the Acme deal."
  → Confirmed. Running close_won workflow: 4 steps, 4 completed.
```

---

## Five Module Domains

### MacroEngine
Global execution state — intent routing, confirmation gates, undo registry.

```
MacroContext FACT → eliza_confidence SCORE → MacroFlow STATE
  idle → awaiting_input (entity ID needed) → confirming → executing → idle
```

Patterns: `what_are_you` ("a very articulate button"), `help`, `undo`, `confirm_yes`, `confirm_no`

Rules: `execute_confirmed_macro`, `block_execution_pending_confirmation`

### AccessControl
Role-based permission enforcement. Permission gates fire before the first workflow step.

```
UserContext FACT (is_manager, is_controller, is_cfo, is_admin)
Rules: require_manager_for_bulk_operations, require_controller_for_month_end,
       require_cfo_for_journal_post, require_admin_for_tenant_ops
```

The operator cannot be talked past these rules. No matter how they phrase the request. No amount of social engineering gets past `PRIORITY 100`.

### CRMEliza
Deal management, customer onboarding, lead qualification.

```
Patterns: close_deal, onboard_customer, qualify_lead, disqualify_lead, schedule_followup
Workflows: close_won (4 steps), onboard_customer (5 steps), qualify_lead (3 steps)
```

"Close the Brightline deal" → stage transition → invoice → project → team notification. Four deterministic steps from one sentence.

### SalesEliza
Quotes, invoices, orders, bulk approvals.

```
Patterns: create_quote, convert_quote, approve_invoice, bulk_approve_invoices, send_invoice
Workflows: create_quote (3 steps), bulk_approve_invoices (3 steps), convert_quote_to_invoice (2 steps)
```

"Approve all invoices under $1k" → fetch pending → bulk approve → notify finance. Manager role required. Automatically enforced.

### ServiceEliza
Ticket triage, escalation, resolution, SLA management.

```
Patterns: create_ticket, escalate_ticket, resolve_ticket, escalate_critical_unassigned, assign_ticket
Workflows: escalate_to_tier2 (3 steps), escalate_critical_unassigned (3 steps)
```

"Escalate all critical tickets waiting more than 2 hours" → fetch → bulk escalate → alert manager. One sentence, zero menu navigation.

### FinanceEliza
Month-end close, journal entries, AR reporting, PO approvals.

```
Patterns: month_end_close, approve_po, generate_ar_report, post_journal_entry, generate_cash_flow
Workflows: month_end_close (5 steps), post_journal_entry (3 steps), approve_purchase_order (3 steps)
```

Month-end close requires Controller role AND explicit confirmation. The RULE fires before step one. The period lock is irreversible and the system knows it.

---

## Twenty Deterministic Workflows

Every workflow is a named recipe. Every step is logged. Every execution is auditable.

| Workflow | Steps | Permission | Confirmation |
|---|---|---|---|
| `onboard_customer` | 5 | staff | no |
| `close_won` | 4 | staff | if high-value |
| `qualify_lead` | 3 | staff | no |
| `disqualify_lead` | 3 | staff | no |
| `create_activity` | 2 | staff | no |
| `create_quote` | 3 | staff | no |
| `convert_quote_to_invoice` | 2 | staff | no |
| `approve_invoice` | 2 | staff | if large |
| `bulk_approve_invoices` | 3 | **manager** | yes |
| `send_invoice` | 2 | staff | no |
| `create_service_ticket` | 3 | staff | no |
| `escalate_to_tier2` | 3 | staff | no |
| `resolve_ticket` | 3 | staff | no |
| `escalate_critical_unassigned` | 3 | staff | yes |
| `assign_ticket` | 2 | staff | no |
| `month_end_close` | 5 | **controller** | yes |
| `approve_purchase_order` | 3 | staff | no |
| `generate_ar_report` | 2 | staff | no |
| `post_journal_entry` | 3 | **CFO** | yes |
| `generate_cash_flow_report` | 2 | staff | no |

---

## Confirmation Gates

Destructive or high-value operations require explicit confirmation before execution:

```
Operator: "Run month-end close."

Eliza: "Running month-end close for May 2026. This will lock the accounting
        period, post depreciation, reconcile GL accounts, and generate the
        trial balance. This cannot be undone. Controller authorization required.
        Confirm?"

Operator: "Yes."

Eliza: "Confirmed. Executing month_end_close: 5 steps."
        Step 1/5 — lock_period ✓
        Step 2/5 — post_depreciation ✓
        Step 3/5 — reconcile_gl ✓
        Step 4/5 — generate_tb ✓
        Step 5/5 — notify_controller ✓
       "Month-end close complete."
```

The confirmation RULE fires at PRIORITY 95. The permission RULE fires at PRIORITY 100. Both must pass before step one runs.

---

## The AI Build-Time Step

`GenerateEliza` is how you extend the macro surface without writing DSL by hand:

```
ACTION GenerateEliza {
  INPUT  sop_documents, role_definitions, workflow_catalog
  OUTPUT patterns_generated, workflows_generated, agi_preview
  AI     "Analyze SOPs. Generate PATTERN declarations for each trigger phrase.
          Generate WORKFLOW declarations for each SOP sequence.
          Generate RULE declarations for each permission constraint.
          Output valid .agi MODULE and WORKFLOW blocks."
}
```

Feed it your operations manual. Every SOP becomes a PATTERN. Every procedure becomes a WORKFLOW. Every policy boundary becomes a RULE. You review a text file, run the test suite, deploy. The AI's contribution is frozen in the `.agi` source — inspectable, reviewable, version-controlled.

---

## Telemetry to OIE

Every macro execution emits a `MacroPacket` to the OIE:

```
MacroPacket { execution_id, operator_id, operator_role, macro_name,
              trigger_phrase, status, step_count, timestamp }
SIGNATURES true    ← every packet cryptographically signed
TTL 365            ← full year retention
```

The OIE surfaces organizational intelligence from Eliza telemetry:

- *"Sarah Kim ran close_won 14 times this month — 3× the team average. Sales process is concentrated."*
- *"bulk_approve_invoices is failing at step 2 in 30% of executions — ERPBulkApproveInvoices is throwing errors."*
- *"month_end_close execution time increased 40% — reconcile_gl step taking 3× longer than baseline."*

Deterministic execution + signed telemetry = organizational intelligence the OIE can actually trust.

---

## Architecture

```
accelerando_eliza.agi
│
├── ENTITY × 3
│   OperatorSession      → one row per operator login
│   MacroExecution       → one row per workflow run (with trigger phrase)
│   MacroStepLog         → one row per step in every execution
│
├── STAGES MacroExecution.status
│   pending → running → completed / failed / aborted
│
├── MODULE × 6              → each governs a domain of the operator surface
│   MacroEngine            → MacroContext FACT, eliza_confidence SCORE, MacroFlow STATE
│   AccessControl          → UserContext FACT, permission RULEs
│   CRMEliza               → 5 PATTERNs, 2 RULEs
│   SalesEliza             → 5 PATTERNs, 2 RULEs
│   ServiceEliza           → 5 PATTERNs, 1 RULE
│   FinanceEliza           → 5 PATTERNs, 3 RULEs
│
├── WORKFLOW × 20           → deterministic multi-step recipes
│   Each: named steps, ON_FAIL (abort or skip), IDEMPOTENT flag
│   Total: 66 workflow steps across 20 workflows
│
├── PACKET MacroPacket      → signed, 365d TTL, feeds OIE
├── CHANNEL eliza_telemetry_out → to accelerando_oie
│
├── ACTION × 32             → ProcessOperatorMessage + ExecuteWorkflow + ERP wrappers
│   ProcessOperatorMessage  → PATTERN match + RULE evaluation (the runtime core)
│   ExecuteWorkflow         → dispatches to WORKFLOW by name, logs every step
│   UndoLastMacro           → reverses completed steps where possible
│   GenerateEliza           → AI build-time: SOPs → PATTERN + WORKFLOW declarations
│   ExplainLastAction       → AI: plain-language explanation of what just executed
│   AuditMacroUsage         → AI: execution pattern analysis + recommendations
│   ERPCreate* / ERPApprove* / ERPGenerate* → cross-app ERP calls via HTTP
│
├── VIEW × 4                → operator console
│   ElizaTerminal           → the conversational interface
│   MacroHistory            → full execution audit table
│   StepAuditLog            → per-step detail view
│   PermissionMatrix        → operator roles and authorization levels
│
└── PREFERENCE × 3          → high_value_deal_threshold, bulk_approval_limit, sla_escalation_hours
```

---

## The Full Accelerando Stack

```
accelerando_erp.agi      → business data (32 entities, 32 actions)
accelerando_chatbot.agi  → customer-facing service (deterministic, cannot hallucinate)
accelerando_eliza.agi    → operator-facing interface (deterministic, macro executor)
accelerando_es.agi       → internal governance (deterministic rules, policy enforcement)
accelerando_oie.agi      → organizational intelligence (AI reasoning over all telemetry)
```

Five `.agi` files. One coherent AI-native enterprise platform.

The chatbot handles customers. Eliza handles operators. The expert system enforces policy. The OIE surfaces intelligence. The ERP stores everything.

None of them trust an LLM at runtime. All of them are auditable. All of them are reproducible.

---

## The Training Manual Question

If every SOP compiles to a PATTERN and every procedure compiles to a WORKFLOW, what is the training manual for?

Nothing. It becomes the build-time input to `GenerateEliza`. You read it once. The compiler reads it forever.
