# Accelerando Chatbot — Super Eliza Customer Service

**A deterministic customer service chatbot that literally cannot go off the rails.**

> "Our chatbot will never call your customer a racial slur."  
> Verified. Deterministic. Auditable.

Every AI chatbot deployed live on the web is a bet that the model behaves this time. Agicore takes a different position: AI runs **once, at build time**, generating a deterministic artifact. Customers interact with the artifact, not the model. The worst case is "I didn't understand that — let me connect you with a person." It cannot hallucinate a 90-day return policy. It cannot be jailbroken into saying something awful. It cannot drift.

---

## The Architecture

```
BUILD TIME (once, with AI):
  Product docs + FAQ + escalation policy
       │
  GenerateChatbot ACTION (AI)
       │
  PATTERN declarations (inspectable, reviewable, version-controlled)
  RULE declarations (safety policies, escalation triggers)
       │
  accelerando_chatbot.agi
       │
  Agicore compiler → Axum + React + deployed artifact

RUNTIME (every customer conversation):
  Customer message
       │
  PATTERN engine (deterministic — match keywords, return template)
       │
  RULE engine (deterministic — safety policies fire before response)
       │
  Response (compiled template) or EscalateToHuman
```

The LLM never touches a live customer. There is no prompt to jailbreak. There is no hallucination surface. The AI did its job at build time; the rules do their job at runtime.

---

## Five Governance Modules

### ConversationEngine
Global conversation state — intent routing, session lifecycle, escalation tracking.

```
ConversationContext FACT → overall_confidence SCORE → ConversationFlow STATE
  greeting → routing (intent detected) → resolving → closed
                                        → escalated (needs_human == true)
```

Rules: `escalate_no_match_loop`, `session_resolved`

### BillingSupport
Billing questions, invoice disputes, payment failures, subscriptions.

```
BillingContext FACT → billing_confidence SCORE → BillingFlow STATE
  5 PATTERNs: billing_general, payment_failed, dispute_charge, invoice_missing, subscription_cancel
  4 RULEs: block_refund_without_verification (PRIORITY 100), escalate_large_dispute (>$500),
           auto_credit_small_dispute (<$50 auto-approved), flag_cancellation_intent
```

Policy encoded: **A refund can never be promised without order verification. Disputes over $500 always go to a human.**

### OrderManagement
Order status, shipping delays, wrong items, returns, cancellations.

```
OrderContext FACT → order_confidence SCORE → OrderFlow STATE
  5 PATTERNs: order_status, shipping_delay, wrong_item, cancel_order, return_request
  4 RULEs: require_order_id_before_status, block_cancel_if_shipped, escalate_delivery_overdue (>14d),
           auto_approve_return (eligible + delivered)
```

Policy encoded: **You cannot cancel a shipped order. Delivery date cannot be promised without the verified order. 14-day delays always escalate.**

### TechnicalSupport
Product bugs, feature questions, integrations, performance issues.

```
TechContext FACT → tech_confidence SCORE → TechFlow STATE
  5 PATTERNs: product_bug, data_loss, feature_question, integration_issue, performance_issue
  3 RULEs: immediate_escalate_data_loss (PRIORITY 100), immediate_escalate_critical_bug (PRIORITY 100),
           create_ticket_when_unresolved
```

Policy encoded: **Data loss escalates immediately, no exceptions. Critical bugs page engineering on-call.**

### AccountManagement
Password reset, email changes, user management, plan changes, account closure.

```
AccountContext FACT → account_confidence SCORE → AccountFlow STATE
  5 PATTERNs: reset_password, change_email, add_user, downgrade_plan, close_account
  3 RULEs: block_account_change_without_identity (PRIORITY 100), escalate_account_closure,
           flag_downgrade_for_retention
```

Policy encoded: **Identity must be verified before any account change. Account closure always goes to a retention specialist.**

### SafetyNet
The module that makes the headline true.

```
SafetyContext FACT → frustration_score SCORE → EscalationFlow STATE
  3 PATTERNs: human_request (PRIORITY 100), frustration_signal, off_topic
  6 RULEs: escalate_human_request (PRIORITY 100), escalate_repeated_no_match,
           never_promise_delivery_without_order (PRIORITY 100),
           never_promise_refund_without_verification (PRIORITY 100),
           escalate_high_frustration, block_off_topic_response
```

These rules fire on every conversation, regardless of which module is active. They are the behavioral floor beneath every response the chatbot can produce.

---

## Why This Beats Every LLM Chatbot

| Concern | LLM Chatbot | Super Eliza / Agicore |
|---|---|---|
| Promises non-existent refund policy | Possible every call | Impossible — policy is in RULE |
| Offensive response | Possible via jailbreak | Impossible — only compiled templates |
| Inconsistent answer | Varies by prompt/context | Same input → same response, always |
| Data loss escalation | "I'll look into that" | PRIORITY 100 RULE fires before response |
| Auditability | "The model said so" | Rule name, FACT snapshot, timestamp |
| Cost at scale | Per-token on every message | Zero marginal cost at runtime |
| Regulatory compliance | Impossible to prove | Trivial to prove |

---

## The AI Build-Time Step

The `GenerateChatbot` ACTION is the multiplier. Feed it your product docs, FAQ, and escalation policy:

```
ACTION GenerateChatbot {
  INPUT  knowledge_docs, escalation_policy, product_name
  OUTPUT patterns_generated, rules_generated, agi_preview
  AI     "Cluster all customer intents. Generate PATTERN declarations for each.
          Generate RULE declarations for each policy boundary.
          Output a valid .agi MODULE block."
}
```

An AI trained on your support history can generate hundreds of PATTERN declarations covering the long tail of customer intents — things your FAQ never explicitly addressed but your support team has answered a thousand times. You review the generated `.agi` text. You run the test suite. You deploy.

The AI's contribution is inspectable, reviewable, and version-controlled. It's not running live. It ran once, produced a text file, and its job is done.

---

## Safety Policies Are Readable

The entire behavioral policy of this chatbot is readable prose:

```
RULE never_promise_delivery_without_order {
  WHEN     ConversationContext.current_intent == "shipping_delay"
  AND      ConversationContext.order_verified == false
  THEN     FLAG "block_delivery_promise"
  SEVERITY critical
  PRIORITY 100
}
```

A non-engineer can read this. A compliance officer can audit it. A lawyer can cite it. Your support manager can change it. None of those people can audit a system prompt.

---

## Telemetry to OIE

Every escalation emits an `EscalationPacket` to the `escalation_out` CHANNEL, which feeds the Accelerando OIE:

```
EscalationPacket {
  session_id, tenant_id, reason, trigger_rule, intent, module_name, timestamp
  SIGNATURES true    ← cryptographically signed
  TTL 90             ← 90-day retention
}
```

The OIE surfaces meta-intelligence: "SafetyNet.escalate_high_frustration fired 340 times this month — up 80% from last month. The BillingSupport module is generating disproportionate frustration signals." That insight cannot come from a black-box LLM chatbot. It can only come from a system where every decision has a name.

---

## Architecture

```
accelerando_chatbot.agi
│
├── ENTITY × 4
│   ConversationSession     → one row per customer chat
│   MessageLog              → every message, speaker, pattern matched
│   EscalationRecord        → every human handoff with trigger rule
│   KnowledgeEntry          → the knowledge base the chatbot draws from
│
├── STAGES ConversationSession.status
│   active → resolved, active → escalated, escalated → resolved
│
├── MODULE × 5              → each has FACT + SCORE + PATTERN + RULE + STATE
│   ConversationEngine      → global state: ConversationContext FACT
│   BillingSupport          → 5 PATTERNs, 4 RULEs, BillingFlow STATE
│   OrderManagement         → 5 PATTERNs, 4 RULEs, OrderFlow STATE
│   TechnicalSupport        → 5 PATTERNs, 3 RULEs, TechFlow STATE
│   AccountManagement       → 5 PATTERNs, 3 RULEs, AccountFlow STATE
│   SafetyNet               → 3 PATTERNs, 6 RULEs, EscalationFlow STATE
│
│   Total: 28 PATTERNs, 20 RULEs, 6 SCOREs, 6 STATE MACHINES
│   (A real deployment would have 200-2,000+ PATTERNs from GenerateChatbot)
│
├── PACKET EscalationPacket → signed, 90d TTL, feeds OIE
├── CHANNEL escalation_out  → to accelerando_erp
│
├── ACTION × 9              → runtime operations + AI build-time tools
│   StartSession            → create session, initialize FACT state
│   ProcessMessage          → PATTERN match + RULE evaluation (the runtime core)
│   CheckOrderStatus        → ERP integration, real order data
│   ProcessReturn           → ERP integration, return label generation
│   ApplyAutoCredit         → auto-approve small billing disputes
│   CreateSupportTicket     → ERP ServiceTicket from conversation
│   EscalateToHuman         → human handoff + EscalationPacket emit
│   EndSession              → close session, persist summary
│   GenerateChatbot         → AI build-time: docs → PATTERN declarations
│   ExplainEscalation       → AI: plain-language escalation explanation
│   AuditSafetyPolicy       → AI: rule firing analysis + recalibration advice
│
├── VIEW × 4                → admin console
│   ChatInterface           → customer-facing chat widget
│   AgentQueue              → kanban of open escalations
│   ConversationHistory     → full audit table of all messages
│   KnowledgeBase           → manage knowledge entries
│
└── PREFERENCE × 3          → escalation_threshold, auto_credit_limit, session_timeout
```

---

## Running with the Full Accelerando Stack

```
accelerando_erp.agi      → business data, order records, billing, service tickets
accelerando_chatbot.agi  → customer service layer (this app)
accelerando_es.agi       → governance layer (internal policy enforcement)
accelerando_oie.agi      → intelligence layer (AI reasoning over all telemetry)

Data flow:
  Customer message → ProcessMessage → PATTERN engine → RULE engine → Response
  Escalation       → EscalateToHuman → EscalationPacket → escalation_out → OIE
  Order check      → CheckOrderStatus → ERP orders table → real data back
  Ticket created   → CreateSupportTicket → ERP ServiceTicket → agent queue
```

Four `.agi` files. One coherent AI-native enterprise platform. One deterministic chatbot that the legal team can actually sign off on.
