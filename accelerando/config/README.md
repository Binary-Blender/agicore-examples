# Accelerando Configuration Intelligence

**The ERP that knows itself. The finite configurations encoded as a deterministic expert system.**

ERP customization *looks* infinite. It isn't.

It's a combinatorial space over ~20 industry verticals × 5 size tiers × 10 regulatory frameworks × 50 workflow patterns. SAP's implementation consultants navigate this space manually, charging $500k and taking 3-6 months. They configure the same 15 things every time for a manufacturing company at your stage. That knowledge is encodable. It is now encoded.

---

## The Two-Phase Model

```
Phase 1 — Initial Setup (AI, one time)

  "We're a 60-person manufacturing company. Government contracts.
   Three warehouses. QuickBooks we've outgrown."
         ↓
   ConductSetupInterview ACTION (AI reads your answers)
         ↓
   MatchConfigurationTemplate (deterministic — 13 known templates)
         ↓
   "Matched: manufacturing_medium + government_contractor overlay.
    Confidence: 94%. 847 companies at your stage use a similar config.
    Modules to add: bom, work_orders, multi_warehouse, cost_accounting,
    far_approval_workflows, dcaa_reporting. Apply?"
         ↓
   ApplyKnownConfiguration (deterministic, no AI)
         ↓
   Accelerando deployed, configured, ready.

Phase 2 — Ongoing Self-Customization (deterministic, ongoing)

  Business state changes:
    "We just crossed 250 employees."
    "We're getting SOX audited next year."
    "We're expanding into Europe."
         ↓
   ConfigurationProfile FACT updated
         ↓
   ES evaluates: which known configurations are now indicated?
         ↓
   RULE fires: recommend_medium_to_large_transition (employee_count > 250)
   RULE fires: sox_gap_journal_approval (sox == true, gap flag set)
   RULE fires: gdpr_gap_erasure_workflow (gdpr == true, gap flag set)
         ↓
   Eliza: "Three configuration changes are recommended. Apply all?"
         ↓
   Operator confirms → ApplyKnownConfiguration × 3 → done.
```

---

## Six Advisory Modules

### ConfigurationEngine
The master profile. Every other module reads and writes `ConfigurationProfile`.

```
ConfigurationProfile FACT — the ERP's self-knowledge:
  industry, employee_tier, revenue_tier, approval_complexity
  has_inventory, has_manufacturing, has_multi_warehouse
  has_subscription_billing, has_project_billing, has_bom_module
  compliance_sox, compliance_hipaa, compliance_government, compliance_gdpr
  active_modules, setup_complete, matched_template_id
```

The FACT is the configuration. The ES reasons over it. Eliza presents recommendations from it. The state of the system is always inspectable.

### IndustryAdvisor
Industry-specific recommendations. Manufacturing needs BOM. SaaS needs subscription billing. Healthcare mandates HIPAA. These are not opinions — they are rules.

```
Industries: manufacturing, saas, professional_services, retail, healthcare, construction
Rules: manufacturing_needs_bom, saas_needs_subscription_module,
       services_needs_time_tracking, healthcare_requires_hipaa (PRIORITY 100)
```

`healthcare_requires_hipaa` fires at PRIORITY 100. It cannot be skipped. If you're in healthcare and HIPAA isn't configured, the system tells you before you touch a patient record.

### GrowthAdvisor
Size-tier transition intelligence. Every company grows through the same inflection points.

| Transition | Headcount | What changes |
|---|---|---|
| Startup → Small | > 10 | First approval chains, expense policy |
| Small → Medium | > 50 | Director approval tier, department budgets |
| Medium → Large | > 250 | Board approval threshold, internal audit, SOX readiness |
| Large → Enterprise | > 1,000 | Full governance stack, compliance mandatory |

The most common configuration update in the entire library: `growth_small_to_medium`. 1,247 companies. Same config. Every time.

### ComplianceAdvisor
Regulatory gap detection. Every gap has a name. Every gap has a deterministic fix.

```
SOX gaps:        sox_gap_journal_approval, sox_gap_4eyes, sox_gap_audit_trail
HIPAA gaps:      hipaa_gap_access_controls, hipaa_gap_baa_tracking
Government gaps: government_gap_cost_accounting, government_gap_far_workflows
GDPR gaps:       gdpr_gap_erasure_workflow, gdpr_gap_data_residency
```

`ScanComplianceGaps` ACTION evaluates every active framework against the current configuration, sets boolean gap flags on `ComplianceContext` FACT, and fires the appropriate RULEs. The compliance scan runs automatically every 30 days (configurable). Gaps are not opinions. They are flags.

### WorkflowAdvisor
Workflow maturity progression. Companies evolve from informal to structured to regulated.

```
ar_overdue_rate > 15% → recommend_credit_controls
employee_tier == "medium" AND no po_approval_chain → recommend_po_approval_chain
approval_avg_days > 5.0 → approval_chain_bottleneck (the chain is the problem)
```

The last rule is the interesting one. The ES can detect that an approval chain you added is creating a bottleneck and recommend restructuring it. The ERP is observing its own workflows.

### IntegrationAdvisor
```
using_quickbooks AND employee_count > 25 → quickbooks_outgrown
using_stripe AND no_invoice_approval → recommend_stripe_ar_automation
```

QuickBooks works until it doesn't. The ES knows exactly when.

---

## The Template Library — 13 Known Configurations

Each template row includes a `usage_count` — how many companies run this configuration. When Eliza recommends `growth_small_to_medium`, she says: *"1,247 companies at your stage use this configuration."* That's not marketing. That's provenance.

| Template | Industry | Tier | Compliance | Users |
|---|---|---|---|---|
| `saas_startup` | SaaS | startup | — | 312 |
| `saas_small` | SaaS | small | — | 847 |
| `manufacturing_small` | Manufacturing | small | — | 523 |
| `manufacturing_medium` | Manufacturing | medium | — | 298 |
| `professional_services_small` | Services | small | — | 634 |
| `professional_services_medium` | Services | medium | — | 211 |
| `retail_small` | Retail | small | — | 445 |
| `healthcare_hipaa` | Healthcare | small | HIPAA | 189 |
| `government_contractor` | Any | any | FAR/DCAA | 156 |
| `sox_overlay` | Any | large | SOX | 94 |
| `gdpr_overlay` | Any | any | GDPR | 273 |
| `growth_small_to_medium` | Any | medium | — | **1,247** |
| `growth_medium_to_large` | Any | large | — | 412 |

The library grows. When `GenerateCustomConfiguration` creates a novel configuration used by 5+ tenants, it gets promoted to the library automatically. The system learns deterministically — not by training a model, but by cataloguing what actually works.

---

## Seven Deterministic Workflows

```
initial_configuration      → 6 steps: intake → match → generate → apply → scan → notify
apply_manufacturing_profile → 5 steps: bom → work_orders → inventory → preferences → log
apply_saas_profile         → 5 steps: subscription → mrr → churn → preferences → log
apply_sox_compliance       → 6 steps: 4eyes → journal_approval → audit_trail → reconciliation → gap_check → log
apply_government_compliance → 5 steps: cost_accounting → far → dcaa → gap_check → log
apply_growth_small_to_medium → 5 steps: director_tier → expense_policy → dept_budgets → update_flag → log
apply_growth_medium_to_large → 6 steps: board_approval → internal_audit → sox_assessment → consolidated → update_flag → log
scan_and_remediate_compliance → 5 steps: scan → report → apply → rescan → notify
```

Every configuration change is a named workflow. Every step is logged. `ConfigurationChange` records the before/after state snapshot. You can answer "what changed, when, who confirmed it, and which template was applied" for any configuration change in the system's history.

---

## The Recursive Property

The configuration advisor is itself a known, finite space. You could build a `config_of_the_config_advisor.agi`.

More practically: the ComplianceAdvisor module that advises on SOX compliance is itself SOX-compliant — every change it recommends is logged, signed, and auditable. The system that enforces the audit trail is itself on the audit trail.

The ERP knows itself. The ES that governs configuration is itself configurable. The Eliza interface that presents recommendations is itself governed by the same RULE engine. It's deterministic turtles all the way down.

---

## Architecture

```
accelerando_config.agi
│
├── ENTITY × 3
│   ConfigurationTemplate   → the known configuration library (13 seed rows)
│   ConfigurationChange     → every configuration change with before/after state
│   ComplianceCheck         → compliance scan results per framework
│
├── STAGES ComplianceCheck.status
│   pending → clean / gaps_found → remediated
│
├── MODULE × 6              → advisory domains
│   ConfigurationEngine     → ConfigurationProfile FACT (master state), SetupFlow STATE
│   IndustryAdvisor         → 7 PATTERNs (one per industry), 5 RULEs
│   GrowthAdvisor           → 5 PATTERNs, 6 RULEs, GrowthFlow STATE
│   ComplianceAdvisor       → 5 PATTERNs, 8 RULEs, compliance_score SCORE
│   WorkflowAdvisor         → 3 PATTERNs, 4 RULEs, workflow_maturity SCORE
│   IntegrationAdvisor      → 3 PATTERNs, 2 RULEs
│
├── WORKFLOW × 8            → configuration application sequences
│   All: deterministic, logged, reversibility-flagged
│
├── ACTION × 12             → setup + matching + compliance + audit
│   ConductSetupInterview   → AI: intake conversation → profile JSON
│   MatchConfigurationTemplate → deterministic: score templates, return best match
│   GenerateInitialConfiguration → AI: novel requirements → DSL modules
│   ApplyKnownConfiguration → deterministic: apply template, update profile, log
│   GenerateCustomConfiguration → AI: requirements outside library → new template
│   ScanComplianceGaps      → deterministic: evaluate every active framework
│   GenerateComplianceReport → AI: plain-language gap report + remediation steps
│   AuditConfigurationHistory → AI: configuration evolution analysis
│
├── VIEW × 4                → configuration console
│   ConfigurationDashboard  → current profile and recommendations
│   TemplateLibrary         → browse and manage known configurations
│   ChangeHistory           → full configuration audit trail
│   ComplianceStatus        → gap status by framework
│
└── PREFERENCE × 3          → auto_apply_known_configs, compliance_scan_interval_days,
                               usage_threshold_for_library
```

---

## The Full Accelerando Stack

```
accelerando_erp.agi       → business data (32 entities, 32 actions)
accelerando_config.agi    → self-configuration (this app — 13 templates, 6 advisory modules)
accelerando_chatbot.agi   → customer service (deterministic, cannot hallucinate)
accelerando_eliza.agi     → operator interface (20 workflows, macro executor)
accelerando_es.agi        → governance (34 rules, 6 policy modules)
accelerando_oie.agi       → organizational intelligence (AI reasoning, retrospective)
```

Six `.agi` files. One coherent AI-native enterprise platform.

The config advisor configures the ERP. The ERP runs the business. The expert system enforces policy. Eliza executes workflows. The chatbot serves customers. The OIE tells you what it all means.

None of them trust an LLM at runtime. All of them know what they did and why.

---

## What Used to Cost $500k

An SAP implementation for a 60-person manufacturing company typically involves:
- 3-6 months of consultant time
- A 200-question discovery process
- Manual configuration of 15-20 standard modules
- A training manual no one reads
- An annual support contract to handle the inevitable misconfigurations

Accelerando Config does this in a conversation. The discovery process is `ConductSetupInterview`. The 15 standard modules are the template library. The training manual is Eliza. The support contract is the ES continuously monitoring for configuration drift and compliance gaps.

The knowledge was never proprietary. It was just expensive to apply manually. Now it's a SEED file.
