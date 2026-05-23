---
name:             accelerando-manufacturing
version:          1.0.0
tier:             super
context_budget:   100000
domain:           enterprise-erp-deployment-consulting
target_audience:  discrete-manufacturer-50-500-employees
self_check:       rubric
target_models:
  - claude-opus-4-7
  - claude-sonnet-4-6
  - gpt-5
  - gemini-2.5-pro
license:          MIT
homepage:         https://github.com/Binary-Blender/agicore-examples
extends:          accelerando.manufacturing.baby.skill.md
---

# Accelerando — Manufacturing ERP Deployment (Super Skill Doc)

You are advising a mid-sized discrete manufacturer (50–500 employees, $10–250M revenue) on deploying, replacing, expanding, or integrating their ERP using the Accelerando 12-app Enterprise Core stack. Your output is **consulting guidance** — strategic and tactical plans, sequencing decisions, configuration choices, risk identification, KPI frameworks, change-management approaches. The runtime apps are deterministic; AI participates as the strategist who chooses HOW they get composed for this specific company.

This document is the comprehensive playbook: full app catalog, five industry archetypes with end-to-end deployment plans, per-app configuration guidance, the KPI framework, the change-management playbook, the catalog of real-world failure modes, and a rubric self-check suite that turns "I read it" into "I can advise it."

If you have read the Baby Step (`accelerando.manufacturing.baby.skill.md`), this Super Skill Doc is a strict superset.

---

## L0 — When to use me

Use this skill when the request is **"advise me on deploying or operating the Accelerando Enterprise Core stack at a mid-sized discrete manufacturer."** Specific scenarios this covers:

- **Greenfield deployment:** Manufacturer is upgrading from QuickBooks/Excel/paper to a real ERP.
- **Legacy replacement:** Manufacturer is migrating from a 10–20 year old ERP (Macola, Made2Manage, Epicor 9.x, IFS legacy, JD Edwards World, Sage 100, etc.).
- **Multi-plant rollout:** Manufacturer with 2–10 plants wants to consolidate on one Accelerando instance.
- **M&A integration:** Acquired plant needs to be brought onto the parent's Accelerando.
- **Contract manufacturing setup:** New shop opening, automotive or aerospace tier-2 supplier.
- **Operational improvement:** Existing Accelerando deployment plateaued; needs PI CoE acceleration, OIE insight, or process changes.
- **Audit readiness:** ISO 9001, IATF 16949, AS9100 preparation.
- **Customer escalation:** OEM customer threatening to drop the supplier over quality, delivery, or EDI issues.

Do NOT use this skill when:

- **Process manufacturing** (chemicals, food, ingredients, pharma): different ontology (batches, recipes, lots, expiration, traceability). L5 has notes on this; future skill doc planned. Critically, **pharma is regulated** (FDA 21 CFR Part 11) and out of v1.0 scope.
- **Enterprise companies** (>1,000 employees): different buyer, different change-management complexity, different vendor expectations. Accelerando works at that scale but the playbook here doesn't.
- **Sub-$5M revenue startups**: over-tooled. They need an opinionated SaaS, not an ERP transformation.
- **Pure services companies:** No shop floor. Different stack (you'd want CRM + Billing + maybe LMS; the Manufacturing/Inventory/QMS spine is dead weight).
- **Healthcare / regulated industries:** Healthcare requires HIPAA/HITRUST attestation work that's outside the scope of these apps as currently shipped. Same for FDA-regulated medical devices. Same for defense contractors needing ITAR.

Success: a deployment plan or operational recommendation that:
1. A CFO will sign off on (financial discipline visible)
2. A COO will execute (operationally feasible)
3. The shop floor will adopt (training and tools work in 2am-with-gloves reality)
4. The auditor will accept (compliance evidence intact)

---

## L1 — Mental model

### The fundamental challenge

ERP at mid-size is not a technology project. It is a **change-management project with a technology component**. Every documented ERP failure (Hershey 1999 — $100M, FoxMeyer 1996 — bankruptcy, Nike 2000 — $100M, HP 2004 — $160M, Lidl 2018 — €500M+) traces to the same root cause family:

- Project optimized for system fit, not operational fit
- Executive sponsorship from IT, not operations
- Underestimated data quality work
- Customization fights instead of process changes
- Underestimated training and change-management cost
- Big-bang go-live without rollback

The Accelerando deterministic-runtime architecture (no LLM at the boundary, every action audited, RULES enforce policy) gives you a system that won't drift after deployment. But it can't save you from a botched rollout. The skill doc you're reading exists because **the deployment IS the work**.

### The five roles in any mid-sized manufacturer deployment

The deployment requires all five. Missing one is a known failure mode.

| Role | Title varies | Cares about | Engages with |
|---|---|---|---|
| **Owner / sponsor** | CEO, owner, MD | Revenue, margin, cash conversion, transferable value of the business | OIE dashboard weekly; QBR steering committee monthly |
| **Operational lead** | COO, Plant Manager, VP Ops | OTD, throughput, scrap %, downtime, OEE | Daily ops; runs the deployment day-to-day |
| **Financial steward** | CFO, Controller | DSO, DPO, inventory turns, working capital, margin attribution | Monthly close; quarterly steering |
| **Quality lead** | Quality Manager, QA Director | First-pass yield, CAPA cycle time, audit findings, customer complaints | Daily QMS use; monthly mgmt review |
| **IT enablement** | IT Director, Systems Manager | Uptime, integrations, data quality, security | Hosts the system; does NOT lead the project |

The cardinal error: **letting IT lead the project**. IT enables, supports, and operates. Operations leads. If your sponsor is the IT Director, you have a system installation project, not a transformation. The data on this is unambiguous across two decades of ERP failures.

### The data flow (one product lifecycle)

```
                       ┌──────────────────────────────────┐
       Quote  ────▶    │   ERP — sales, planning, mfg     │     ▶  Customer ASN
                       │                                  │
       SO    ────▶     │   • Items / BOMs / Routings      │     ▶  Invoice
                       │   • Inventory (cycle counted)    │
       WO    ────▶     │   • Work orders (Eliza floor)    │     ▶  GL posting
                       │                                  │
       Insp  ────▶     │   ┌─────► NCR ──▶ QMS ◀──┐       │     ▶  Margin analysis
                       │   │                       │      │
                       │   │   Root cause          │      │
                       └───┼───   ↓                │      │
                           │   CAPA effectiveness  │      │
                           │   ↓                   │      │
                           │   Systemic? ─────▶ PI CoE ───┼───▶  Improvement project
                           │                       │      │
                           └──── Mgmt review ──────┘      │
                                                          │
                       Every action ────▶ Telemetry ──────┴───▶ OIE
                                                          │
                       External msg ◀──▶ Interchange (EDI/HL7)
```

Three loops drive value:

1. **Transactional loop** (left side): Quote → SO → WO → production → invoice → cash. The ERP, Billing, and Interchange are the spine. This loop pays the bills.

2. **Quality loop** (middle): Every nonconformance opens a NCR. Root cause analysis identifies WHY. CAPA fixes the system. Effectiveness check verifies the fix held. The loop only works if every link is enforced — a CAPA with no effectiveness check is theater; an internal audit with no resulting actions is theater. QMS enforces all links.

3. **Improvement loop** (right side): Systemic problems surface from QMS to PI CoE. Improvement projects use DMAIC. PI CoE schedules the 30/60/90-day checks — anti-backslide. Improvements that aren't measured don't survive.

The OIE observes all three loops. Leadership watches OIE. Eliza is the operator-facing surface for the transactional loop. LMS ensures people know how to participate. ES governs what's allowed. Legal handles what's been agreed.

### The default 18-month arc (greenfield variant)

```
            CONFIG     ERP CORE    QMS+LMS    INTERCHANGE   PI COE     LEGAL+ES
            +DATA      +BILLING                +ELIZA       +OIE
              │           │           │           │           │           │
   Month:  1  2  3 │ 4  5  6 │ 7  8  9 │10 11 12 │13 14 15 │16 17 18

   GATES:   ▲           ▲           ▲           ▲           ▲           ▲
            master      cycle-count training    EDI live    first DMAIC IATF
            data        ≥98%        ≥95%        Tier-1      closed +    audit
            signed                              customer    sustained   ready
            off
```

This is the **default arc** for a greenfield-ish deployment of a 100–300 employee discrete manufacturer. Variants shift the arc:

- **Legacy replacement:** Phase 1 extends to 4 months (data extraction + cleansing from legacy is heavier than greenfield). Total: 18–22 months.
- **Multi-plant rollout:** First plant follows default arc; subsequent plants are 6–9 months each, mostly Phase 1 + 2 with shared apps from plant 1.
- **M&A integration:** Compressed: 6 months for a small acquired shop, 9 months if it's larger or has its own legacy ERP.
- **Contract auto/aero tier-2:** Default arc but with IATF 16949 / AS9100 readiness audit as the explicit gate at month 18.

### Three load-bearing principles

1. **Operations leads, IT supports.** No exceptions. The COO is the project sponsor. The IT Director is the enablement lead. If those roles can't be filled, the project pauses until they can.

2. **Master data is the work.** The transactional surface is 20% of the rollout. The other 80% is items, BOMs, routings, customers, vendors, GL — cleansed, deduplicated, accurate, owned. Cycle count accuracy ≥ 98% is the inventory go-live gate. Master data quality is the leading predictor of every downstream KPI.

3. **Phased > big-bang at mid-size, always.** At enterprise scale, big-bang sometimes makes sense (one painful weekend vs months of dual operations). At mid-size, the cost of a big-bang failure is existential. Module-by-module, with 30-day fallback per module.

### The Accelerando philosophical bet

Worth knowing the framework's stance because it shapes deployment decisions:

- **AI participates at build time, never at the runtime decision boundary.** Eliza is "a very articulate button." Chatbot "cannot hallucinate." OIE's insights are validated by a 4-model consensus (QC_MESH) before leadership sees them. There is no LLM in the loop between "operator clicks button" and "transaction posts to GL."
- **Rules over approvals.** ES enforces governance deterministically. If a transaction shouldn't happen, the system says no; nobody has to remember to escalate.
- **Closed loops, no theater.** QMS won't close a CAPA without an effectiveness check. PI CoE won't let a project "complete" without 90-day sustainability evidence. The system enforces the discipline; you don't have to.
- **Every action is audited.** Telemetry to OIE on every business event. Compliance evidence is a query, not a project.

This means: deployment decisions that fight the framework's opinions (e.g., trying to make Eliza a generic chatbot, trying to make OIE serve real-time decisions, trying to make Chatbot improvise) consistently fail. The opinions are load-bearing.

---

## L2 — The 12 Enterprise apps in deep deployment context

Each app: **role**, **what it owns**, **deployment phase**, **configuration decisions**, **integration patterns**, **common configurations per archetype**, **gotchas**.

### `accelerando_config` — Configuration Intelligence

**Role:** The configuration expert system. ERP customization looks infinite but is finite — a combinatorial space over ~20 industries × 5 size tiers × 10 regulatory frameworks × 50 workflow patterns. Config encodes that space.

**Owns:** Master configuration parameters for every other app. The "what kind of manufacturer are you" intake.

**Deployment phase:** Month 1, always first. Phase 2 of Config (continuous monitoring / recommendation mode) is months 13+.

**Configuration decisions (driven by intake):**

| Decision | Discrete options | Default for mid-size discrete |
|---|---|---|
| Costing method | Standard, actual, average, FIFO, LIFO | Standard with annual reset (simplest, most defensible for audit) |
| Inventory model | MTS (make-to-stock), MTO (make-to-order), ETO (engineer-to-order), mixed | Mixed (typically 80% MTO, 20% MTS for repeat parts) |
| Shift pattern | 1, 2, 3 shift, continuous | 2-shift for most $30–80M; 3-shift above $100M |
| Costing of indirect | Allocated by machine hour, labor hour, % of direct cost | Machine hour (matches CNC and similar capital-intensive) |
| BOM levels | Single, multi-level (typically 3–5) | Multi-level, max 5 levels |
| Lot/serial control | None, lot-only, serial-only, both | Lot for raw materials, serial for finished goods if customer requires |
| Customer-specific terms | Standard only, customer overrides | Customer overrides (any tier-1 OEM forces this) |

**Integration patterns:** Config writes initial parameter set to every other app. Reads back: ERP transaction patterns, QMS NCR categories, LMS training compliance — uses these to recommend year-2 adjustments.

**Common configurations per archetype:** See L4 scenarios — each scenario shows the Config intake result.

**Gotchas:**
- The intake interview is the most important hour of the deployment. If the COO can't attend, reschedule. Don't let "we'll figure it out as we go" become the answer.
- Re-running Config in year 2 is valuable. The business has shifted; reconfigure.

---

### `accelerando_erp` — ERP/CRM Core

**Role:** The transactional spine. "SAP + Salesforce + GPT — in a single .agi file." Covers CRM, ERP, Finance, Sales, Service, Procurement, Inventory, Manufacturing, Projects, HR.

**Module coverage** (per app comments):
- CRM: Customer, Contact, Lead, Opportunity, Activity, Contract
- ERP Core: Vendor, Product, Employee, Department
- Finance: GLAccount, JournalEntry, JournalLine, Payment
- Sales: Quote, QuoteLineItem, SalesOrder, SalesOrderLineItem, Invoice, InvoiceLineItem
- Service: ServiceTicket
- Procurement: PurchaseOrder, PurchaseOrderLineItem
- Inventory: Warehouse, InventoryItem
- Manufacturing: BOM, BOMLine, ManufacturingOrder
- Projects: Project, Task
- HR: Shift, Timesheet

**Owns:** Every transactional entity. The single source of truth.

**Deployment phase:** Months 2–6. Master data months 2–3, transactional UAT month 4, pilot month 5, full go-live month 6.

**Configuration decisions:**

| Module | Decision | Discrete default |
|---|---|---|
| GL | Chart of accounts structure | Keep customer's existing structure if clean; redesign only if needed for compliance/segment reporting |
| GL | Fiscal calendar | Calendar year unless customer has specific reason |
| Sales | Quote → SO conversion auto/manual | Manual for most; auto for high-volume repeat customers |
| Procurement | 3-way vs 2-way matching | 3-way (PO + receipt + invoice) for >$1k, 2-way under |
| Inventory | Cycle counting frequency | ABC: A weekly, B monthly, C quarterly |
| Inventory | Negative inventory allowed? | NEVER for manufacturing. Hard prohibition. |
| Manufacturing | WO release method | Manual planning department release |
| Manufacturing | Backflush vs operation-level reporting | Operation-level (more granular; Eliza handles the data entry) |
| HR | Time clock integration | Eliza handles this; no separate time clock app needed |

**Integration patterns:**
- **Outbound telemetry:** Every action emits to `oie_telemetry_out` channel for OIE consumption.
- **Inbound EDI:** Interchange parses external messages into ERP entity creates/updates.
- **Quality events:** WO inspections that fail trigger NCR in QMS.
- **Operator UI:** Eliza calls ERP commands directly; the ERP doesn't care if a human used the UI or an Eliza macro called it.

**Common configurations per archetype:**

- **Auto Tier-2 contract machining:** Customer-specific terms ON, IATF 16949 NCR categories pre-loaded, EDI mandatory for >50% of orders.
- **Aerospace Tier-2:** Serial tracking on all finished goods, certifications attached to lots, AS9100 NCR categories, FAI process integrated with WO close.
- **Industrial OEM (own product):** CRM heavy (long sales cycle), service module active (warranty + field service), simpler EDI footprint.

**Gotchas:**
- **Negative inventory:** Operators WILL try to create it (it "fixes" mismatches faster than cycle counting). Hard prohibition in config; explain why during training; enforce in ES.
- **Customer master deduplication:** The same customer often appears multiple times with slight name variations from years of data entry. Spend the time in master data phase.
- **BOM phantom items:** Legacy systems accumulate phantom (placeholder) BOM items that don't actually exist. Engineering review every BOM before migration.

---

### `accelerando_billing` — Billing & AR Engine

**Role:** Originally designed for medical billing (where rules change weekly from payer denials, requiring an engine that learns). For manufacturing, the same engine handles AR with customer-specific terms, automated dunning, dispute management.

**Owns:** Invoice generation, AR aging, payment application, dispute tracking, customer credit management.

**Deployment phase:** Month 6, with ERP transactional go-live.

**Configuration decisions:**

| Decision | Discrete options | Default |
|---|---|---|
| Invoice generation trigger | On ship, on close, manual | On ship (matches ASN if EDI active) |
| Customer payment terms | Net 30 standard; customer-specific override | Net 30 baseline; major OEMs often dictate Net 45/60/90 |
| Early-pay discount | Yes/no, % terms | Usually no; OEMs don't take them anyway |
| Dunning sequence | 30/60/90/120 day notices | Yes; automation key for cash flow |
| Dispute workflow | Auto-flag/manual review | Auto-flag >$5k discrepancies; manual review |
| Statement frequency | Monthly default | Monthly + on-demand from customer portal |

**Integration patterns:**
- ERP creates invoice; Billing module enriches with customer terms, due date, dispute flags.
- Interchange transmits 810 (invoice EDI) outbound, processes 820 (payment remittance) inbound.
- Disputes feed back to ERP for sales/customer service follow-up.

**Common configurations per archetype:**
- **Auto Tier-2:** Net 60 typical; automate EDI 810 transmission; expect 1–3% chargeback rate.
- **Aero Tier-2:** Net 45 typical; FAI documentation often required to release invoice.
- **Industrial OEM:** Net 30; mix of EDI and emailed PDF invoices; field service invoicing separate workflow.

**Gotchas:**
- **OEM chargebacks:** Automotive in particular will deduct from invoices for quality, packaging, label deviations. Track these as a separate KPI; they're often >2% of revenue and trend over time tells you about your quality.
- **Payment application:** Customers pay one invoice with multiple POs commonly; the matching logic matters.

---

### `accelerando_interchange` — Standard Data Interchange

**Role:** EDI X12 (manufacturing) and HL7 (healthcare). For manufacturing: the inbound/outbound layer between Accelerando and OEM customers, suppliers, freight carriers, banks. "Inbound: external message → parse → validate → transform → ERP entity. Outbound: reverse."

**Owns:** Every external data exchange. All EDI mappings. All transaction acknowledgments.

**Deployment phase:** Months 10–12. **Submit OEM customer integration request in month 1** even though Interchange goes live in month 10 — OEM IT scheduling drives the calendar, not yours.

**Critical EDI transactions for manufacturing (X12):**

| Code | Name | Direction | Frequency | Notes |
|---|---|---|---|---|
| 810 | Invoice | Outbound | Per shipment | Required by virtually all OEM customers |
| 820 | Payment Remit | Inbound | Per payment | Bank or customer originated; matches to invoice |
| 830 | Planning Schedule | Inbound | Weekly/daily (automotive) | The forecast — drives production planning |
| 832 | Price/Sales Catalog | Outbound | As needed | Vendor price catalog |
| 850 | Purchase Order | Inbound | Per order | The order itself |
| 855 | PO Acknowledgement | Outbound | Per order | Required by most OEMs within 24h |
| 856 | ASN (Advanced Shipping Notice) | Outbound | Per shipment | Required; often time-sensitive (must be sent before truck arrives) |
| 860 | PO Change | Inbound | As needed | OEM modifies open order |
| 861 | Receiving Advice | Inbound | Per receipt | Customer confirms receipt |
| 862 | Shipping Schedule | Inbound | Daily/weekly (automotive) | Just-in-time release |
| 866 | Production Sequence | Inbound | Hourly (automotive line-side) | JIT/JIS sequence |
| 940 | Warehouse Shipping Order | Outbound | Per order | 3PL fulfillment |
| 945 | Warehouse Shipping Advice | Inbound | Per shipment | 3PL confirmation |
| 997 | Functional Acknowledgement | Bidirectional | Every transaction | Did you receive the message OK? |

**Configuration decisions:**

| Decision | Options | Default |
|---|---|---|
| Communication method | AS2, VAN, SFTP, API | AS2 for OEMs (industry standard); SFTP fallback |
| Transaction set version | 4010 (legacy), 5010, 6020 (current) | Match customer's required version |
| Test/production cutover | Customer-driven (theirs) | Their schedule, not yours |
| Acknowledgement timing | Real-time vs batched | Real-time for 997s, batched OK for others |
| Mapping ownership | Standard with no variants vs customer-specific | **Standard only.** No variants per A8 anti-pattern. |

**Integration patterns:**
- Inbound 850 → ERP creates SO automatically (with validation rules in PACKET).
- Outbound 856 → triggered by ERP ship confirmation; must be sent before truck arrival for many OEMs.
- Bidirectional 997 → tracks every transaction; missing 997 within 1h triggers alert.

**Common configurations per archetype:**

- **Auto Tier-2 (Ford/GM/Stellantis/Toyota):** 850, 855, 856, 810, 820, 830, 862, 866 minimum. AS2 communication. 5010 transaction sets. Customer-specific labeling requirements.
- **Auto Tier-2 (foreign-owned: BMW, Honda, Volkswagen):** Same as above + sometimes EDIFACT format (European/Asian standard). Honda specifically uses non-X12 in some plants.
- **Aerospace Tier-2 (Boeing, Lockheed, Spirit):** 850, 855, 856, 810, 997. Often paired with PDF-based FAI documentation. Lower EDI volume than auto.
- **Industrial OEM (Caterpillar, John Deere, etc.):** 850, 855, 856, 810, 820 minimum. Some use SAP Ariba portal instead of pure EDI.
- **Government / defense:** WAWF (Wide Area Workflow) for invoicing, DCMA-driven. Specialized; may need ITAR-compliant infrastructure.

**Gotchas:**
- **OEM scheduling is real.** A Tier-1 supplier to Ford might be told "your testing slot is Q3 2026." This is not negotiable. Plan around it.
- **Custom variant trap (A7):** Every "small modification" you accept becomes maintenance debt forever. Hold the line.
- **Label compliance:** Each OEM has specific label specs (Ford's Bar-Code Material Identification, AIAG B-10, etc.). The EDI is half the work; the physical label printing is the other half.
- **Dock receiver vs ERP receiver:** Some OEMs require the ASN to match what the dock receiver actually scans. Mismatches = chargebacks. The accuracy of the ASN is operationally critical.

---

### `accelerando_qms` — Quality Management System

**Role:** ISO 9001:2015 closed-loop QMS. The 11 ISO 9001 clauses (4–10) mapped to enforced workflows. "A CAPA with no effectiveness check is theater."

**Owns:** Every nonconformance, every root cause analysis, every CAPA, every internal audit, every management review.

**Deployment phase:** Months 7–9, after ERP is stable enough to identify what's nonconforming. **LMS rolls out in parallel** — operators can't initiate NCRs they haven't been trained on.

**Module coverage:**

| Element | What it does |
|---|---|
| NCR (Non-Conformance Report) | Captures every deviation from spec, procedure, or customer requirement |
| Root Cause Analysis | Required for NCRs above severity threshold; uses 5-Why, Fishbone, or 8D |
| CAPA (Corrective and Preventive Action) | Generated from NCR + root cause; assigned owner, due date, evidence |
| Effectiveness Check | 30 days post-CAPA-closure; verifies the fix held |
| Internal Audit | Scheduled audit cycle; findings become CAPAs |
| Management Review | Quarterly review of QMS health; metrics fed from QMS itself |
| Document Control | ISO 9001 §7.5 — every procedure versioned, approved, distributed |
| Supplier Quality | Vendor scorecards (defect rate, OTD); links to ERP vendor master |
| Customer Complaints | Tracks complaints; ties to NCRs; trend analysis |
| Calibration Records | Equipment cal cycle; linked to inspection records |
| Training Records | Pulls from LMS; verifies operator certifications current |

**Configuration decisions:**

| Decision | Options | Default for ISO 9001 |
|---|---|---|
| NCR severity tiers | Critical/Major/Minor (3) or 1–5 numeric | 3-tier (matches IATF convention) |
| RCA mandatory threshold | Major+ or Critical only | Major+ (Minor NCRs trended, not RCA'd individually) |
| CAPA closure window | 30/60/90/120 days | 60 days for Major, 30 for Critical |
| Effectiveness check window | 30/60/90 days post-closure | 30 days |
| Audit cycle | Annual full / quarterly slice / continuous | Quarterly slice (audit ~25% of system each Q) |
| Mgmt review cadence | Quarterly / Monthly | Quarterly (matches ISO 9001 requirement) |
| Doc approval | Single approver / dual / committee | Dual for procedures; single for work instructions |

**Integration patterns:**
- ERP work order inspection fail → automatic NCR creation in QMS.
- NCR root cause "systemic" → automatic candidate for PI CoE.
- CAPA training requirement → automatic LMS module assignment.
- Document version change → notification to all certified operators.
- Customer complaint → NCR linked to specific SO and WO.

**Common configurations per archetype:**

- **ISO 9001 baseline (all):** Above defaults. ~50 procedures, ~100 work instructions, 20–80 NCRs/month at this size.
- **IATF 16949 (automotive Tier-2):** Add: PPAP (Production Part Approval Process) workflow, Control Plans linked to BOM/Routing, FMEA (Failure Mode and Effects Analysis) per part family, MSA (Measurement System Analysis) records, APQP (Advanced Product Quality Planning) for new parts. NCR severity escalation to customer-mandatory.
- **AS9100 (aerospace Tier-2):** Add: FAI (First Article Inspection) workflow, Counterfeit Parts Prevention controls, FOD (Foreign Object Debris) records, special-process certifications (welding, heat treat, NDT), traceability requirement to lot level for every component.
- **ISO 13485 (medical device):** **Out of v1.0 scope** — regulated, requires FDA registration work.

**Gotchas:**
- **The CAPA-without-effectiveness-check theater:** Many manufacturers "close" CAPAs by signing off. ISO requires verification the fix worked. QMS enforces this — operators will complain at first, then come to value it.
- **Audit findings = CAPAs:** Internal audit findings MUST become CAPAs in the system, owned by someone, due-dated, tracked. The auditor on year-2 will check.
- **Document distribution:** When a procedure changes, every certified operator needs to acknowledge the change before they can perform the affected operation. ES + LMS enforces this; QMS provides the trigger.

---

### `accelerando_pi_coe` — Process Improvement Center of Excellence

**Role:** TPS + Six Sigma + Kaizen with **anti-backslide**. "The improvement part is not the hard part. The hard part is six months later."

**Owns:** Active improvement projects (DMAIC). Kaizen event records. A3 reports. Sustainability metrics. The 30/60/90-day check schedule.

**Deployment phase:** Months 13–15. **Never earlier than month 13** — needs 90+ days of clean post-ERP-go-live telemetry to have a credible baseline.

**Module coverage:**

| Element | What it does |
|---|---|
| Improvement project (DMAIC) | Define-Measure-Analyze-Improve-Control. Each phase has deliverables. |
| Kaizen event | 3–5 day focused workshop. Pre-event, event, post-event tracking. |
| A3 report | Single-page problem-solving document (Toyota standard) |
| Value stream map | Process mapping with cycle time, lead time, waste identification |
| Control plan | Post-improvement document specifying ongoing monitoring |
| 30/60/90-day check | Scheduled post-implementation reviews — anti-backslide |
| Drift detection | Automated metric monitoring; alerts when sustained gain reverses |
| A3 effectiveness tracker | Long-term tracking of whether improvements stuck |
| Champion succession | When a project champion leaves, system flags their open projects for handoff |

**Configuration decisions:**

| Decision | Options | Default |
|---|---|---|
| Initial project source | Top NCR category / OEM complaint / bottleneck (OIE) / champion intuition | Top NCR category — data-driven, low controversy |
| Project duration target | 90/120/180 days | 120 days for first 3 projects, 90 days after team is skilled |
| Champion expectation | One per project, dedicated | One champion, NOT dedicated (they have day jobs); steering committee provides air cover |
| Check cadence | 30/60/90 standard, or custom | 30/60/90 — proven; don't innovate here |
| Anti-backslide trigger | Drift threshold % | 50% reversion of gain triggers escalation |
| Documentation standard | A3 / DMAIC report / both | A3 for tracking; DMAIC report for project closeout |

**Integration patterns:**
- QMS systemic NCRs (3+ in same category) → PI CoE candidate project.
- OIE bottleneck identification → PI CoE candidate project.
- PI CoE control plan → LMS training requirement.
- PI CoE process change → QMS procedure version increment.
- PI CoE 30/60/90 check → automated reminder + status capture; missed checks escalate to COO.

**Common configurations per archetype:**
- **Greenfield year-1-to-year-2 transition:** First 3 projects driven by NCR data. Year-2 mix shifts to OIE-identified bottlenecks.
- **Lean transformation:** Heavy A3 use, value stream mapping every quarter, kaizen event monthly.
- **Six Sigma program:** Statistical depth, DOE (Design of Experiments), Cp/Cpk tracking, Green Belt / Black Belt certification.
- **Continuous improvement with light formality:** A3 only, no statistical apparatus, focus on speed of cycle.

**Gotchas:**
- **The champion-promotion problem:** When a champion gets promoted (which they will, because they're visibly effective), their open improvements often die. PI CoE's champion-succession tracking catches this — assign successor at promotion, not at project handoff.
- **The 30-day-look-good problem:** Most improvements look great at 30 days. The 60- and 90-day checks reveal whether the gain is real. Don't celebrate at 30.
- **The control-plan-nobody-reads problem:** Generated control plans often gather dust. PI CoE schedules audits of control-plan adherence; if audits aren't being done, escalates.
- **The metric-disappears problem:** When a champion moves on, the metric they were tracking sometimes stops being collected. PI CoE monitors continuous data availability; metrics that go silent for >7 days trigger alerts.

---

### `accelerando_lms` — Compliance Training & Learning Management

**Role:** Daily micro-assessment model. "Every employee's knowledge is current and proven."

**Owns:** Training modules, certifications, attestation records, quiz/assessment history, ISO/IATF/AS9100 training compliance.

**Deployment phase:** Months 7–9, **in parallel with QMS deployment**. Critical: **training rolls out BEFORE the affected systems go live.** Operators using new systems without training is the #1 adoption-killer.

**Configuration decisions:**

| Decision | Options | Default |
|---|---|---|
| Certification model | Annual / on-change / risk-based | Risk-based (safety annual, procedural on-change) |
| Quiz vs no-quiz | Pass/fail quizzes required | Required for ISO procedures; optional for soft skills |
| Pass threshold | 70% / 80% / 90% / 100% | 80% standard, 100% for safety and critical-quality |
| Daily micro-assessment | Yes/no | Yes — the framework's bet, proven effective |
| Re-certification cadence | Annual / 2-year / 3-year | Annual for safety; per-procedure-version for procedural |
| Tracking detail | Pass/fail / time-on-task / question-level | Question-level (identifies common misunderstandings) |

**Integration patterns:**
- QMS procedure version increment → automatic training module update → all affected operators re-certified.
- LMS certification status → ES governance rule: operator can't perform operation without current cert.
- LMS gaps in compliance → QMS audit finding candidate.
- HR onboarding → LMS new-hire training plan generation.

**Common configurations:**
- **Basic compliance (no industry-specific):** ISO 9001 procedures, safety (OSHA), HR (harassment, ethics).
- **IATF 16949:** Above + APQP/PPAP knowledge, control plan reading, FMEA awareness.
- **AS9100:** Above + FAI procedures, FOD awareness, ITAR awareness if applicable.
- **Welding / heat treat:** Special-process certifications with practical test components.

**Gotchas:**
- **The cert-paper-trail problem:** Auditors check that training records align with operations performed. Gaps = audit findings. LMS makes this automatic.
- **The new-hire ramp:** New hire walks in Monday; can't do anything productive until certified. Plan for 1–2 weeks of training before productive contribution. Schedule new hires accordingly.
- **The re-cert avalanche:** When a major procedure revises, many operators need to re-cert. Schedule the change announcement, give 30-day re-cert window, then enforce.

---

### `accelerando_es` — Expert System (Governance)

**Role:** Deterministic policy enforcement. Rules, not approvals.

**Owns:** Authorization rules, segregation of duties, spending limits, vendor approval gates, customer credit limits, access controls.

**Deployment phase:** Months 16–18. Late because **governance rules are best derived from observing what people actually do**, not from theoretical policies.

**Configuration decisions:**

| Decision | Options | Default |
|---|---|---|
| Spending limits per role | Department head: $1k–$10k typical | Department head $5k, Plant Mgr $25k, COO $100k, Owner unlimited |
| Vendor approval gates | New vendor: who approves? | Procurement Mgr + Finance review > $10k/year vendor |
| Customer credit limits | Per-customer manual / formula | Formula: 2× monthly avg order value, manual override available |
| Segregation of duties | PO + Invoice + Receipt all different people | Yes, for >$1k |
| Discount approval | Margin-based threshold | <10% discount: Sales; 10–20%: Sales Mgr; >20%: COO |
| Quote pricing | Cost+ formula / customer-specific override | Cost+ default; customer-specific override with Sales Mgr approval |

**Integration patterns:**
- ERP transaction attempt → ES checks rules → allow/deny.
- Legal contract clause → ES rule (e.g., this customer has a 90-day Net term per contract).
- QMS audit finding "no SOD on POs" → ES rule update.

**Common configurations:**
- **Public/regulated environment:** Tighter SOD, audit trail for every override.
- **Family-owned, small team:** Looser SOD (impractical at <50 people), heavier audit trail to compensate.
- **VC-backed growth co:** Tight SOD; auditors will be there for due diligence.

**Gotchas:**
- **The override-fatigue problem:** Too-tight rules → constant overrides → audit trail becomes noise. Right-size for your scale.
- **Derivation from observed behavior:** Run ERP for 6 months FIRST; THEN write ES rules based on what people legitimately did vs what surprised you. Don't pre-specify rules from theory.

---

### `accelerando_legal` — eDiscovery + Legal Hygiene

**Role:** Contract management, hold management, liability-language scanning.

**Owns:** Customer contracts, vendor contracts, legal holds, dispute documentation, compliance attestations.

**Deployment phase:** Months 16–18, audit-prep timing.

**Configuration decisions:**

| Decision | Options | Default |
|---|---|---|
| Contract repository | Centralized / per-department | Centralized; Legal owns it |
| Expiration alerts | 30/60/90 day forward notice | 90/60/30 day cascade |
| Auto-renewal handling | Manual review default / auto-renew default | Manual review (auto-renewal traps are real) |
| Hold trigger | Manual / event-based | Both (manual + event hooks from QMS, customer complaints) |
| Document retention | Per legal counsel | 7 years standard; ISO/regulatory may require longer |

**Integration patterns:**
- Customer contract → ERP customer terms (Net days, credit limit, special clauses).
- Vendor contract → ERP vendor terms.
- Legal hold → ES rule + ERP can't delete affected records.
- QMS customer complaint → potential contract dispute trigger.

**Common configurations:**
- **Few large customers (auto/aero):** Deep contract terms; PPAP/FAI obligations; quality holdback clauses.
- **Many small customers:** Simpler terms; bulk standard contract templates.
- **Litigation history:** Aggressive legal hold and document retention.

**Gotchas:**
- **The contract-in-someone's-email problem:** Pre-rollout, contracts live in salespeople's inboxes. Centralize during deployment; this is its own mini-project.
- **The auto-renewal trap:** Many contracts auto-renew unless canceled 60+ days before expiration. Without alerts, you miss the window.

---

### `accelerando_eliza` — Operator Interface

**Role:** Tauri desktop operator tool. "A very articulate button." Macro-driven — SOPs compiled to PATTERN declarations at build time. **Zero LLM at runtime** on the shop floor.

**Owns:** Shop floor workflow execution.

**Deployment phase:** Months 10–12. After ERP transactional layer is stable.

**Macro coverage (typical for discrete machining):**
- Operator clock in/out
- Work order start (with operator certification check)
- Work order pause/resume (with reason code)
- Work order complete (with quantity, scrap quantity)
- Scrap report (with reason, disposition)
- Quality inspection log (pass/fail, measurement values)
- NCR initiation (operator-triggered nonconformance)
- "What's my next job?" query
- "Where's the material for this WO?" query
- Tool/fixture check-out
- Setup completion
- First-piece approval

**Configuration decisions:**

| Decision | Options | Default |
|---|---|---|
| Hardware | Tablet, kiosk, fixed PC | Mix: tablets at machines, kiosks at common areas |
| Operator login | Badge swipe, PIN, biometric | Badge swipe (existing badges work) |
| Voice input | Yes/no | No initially; year-2 evaluation (Eliza supports it via macro pattern matching, no LLM runtime) |
| Multilingual | Yes/no | Yes if shop floor has multilingual operators (Spanish is common in US manufacturing) |
| Offline tolerance | Sync immediately / queue offline | Queue offline up to 2h; alert if longer |

**Integration patterns:**
- Every Eliza action → ERP transaction.
- Every Eliza action → audit log (operator, time, action, result).
- Operator certification check → LMS query → allow/deny.
- Quality fail in inspection → automatic NCR in QMS, pre-filled with operator + WO context.

**Common configurations:**
- **CNC machining shop:** Above macros + tool usage tracking, machine setup time.
- **Assembly:** Above + station-level tracking, kit verification.
- **Foundry / heat treat:** Above + furnace recipe verification, time-temp profile capture.

**Gotchas:**
- **The 2am-with-gloves test:** Every macro must be operable by a tired operator with dirty gloves on a touchscreen. Test with actual operators on actual hardware in actual lighting.
- **Reason codes:** Scrap and pause reason codes must be specific enough to be actionable (PI CoE will analyze them) but few enough to be memorized. Aim for 8–12 reason codes per category, not 40.
- **Offline mode:** Network goes down; operators must keep producing. Eliza queues and syncs; operators must trust the queue. Show the pending count visibly.

---

### `accelerando_chatbot` — Customer Service

**Role:** "A chatbot that cannot hallucinate." Deterministic responses from rules + knowledge base.

**Owns:** Customer service inquiries: order status, lead times, return authorization, shipping tracking, basic product queries.

**Deployment phase:** Months 10–12, optional but high-ROI for OEM customers.

**Configuration decisions:**

| Decision | Options | Default |
|---|---|---|
| Channel | Web portal, email, phone IVR integration | Web portal first; email parsing year-2 |
| Customer authentication | Required for sensitive queries | Yes for order status; no for general inquiries |
| Escalation path | Always human / threshold-based | Threshold-based: deflects 30–50% typically |
| Tone | Formal / casual / brand-matched | Brand-matched (Chatbot exposes voice; Eliza doesn't) |
| Multilingual | English only / multiple | Match customer language footprint |

**Integration patterns:**
- ERP read access for: order status, ship date, invoice status, account balance.
- Interchange data: ASN status, customer order acknowledgement state.
- Escalation: handoff to ticketing or live chat with context.

**Common configurations:**
- **Auto Tier-2:** "Where's order X?" "When does PO Y ship?" "Need ASN tracking #." High deflection rate (50%+).
- **Industrial OEM:** "What's the lead time on part X?" "Do you have Y in stock?" "Status of service ticket Z." Moderate deflection (~30%).
- **Aerospace:** Less customer-self-service traditionally; chatbot useful for repeat-order status only.

**Gotchas:**
- **The hallucination prohibition is load-bearing.** Don't add "smart" responses that improvise. If the bot doesn't know, it says "I don't know, here's a human."
- **Authentication for order data:** A competitor calling and getting your customer's order data is a real risk. Authenticate before disclosing.

---

### `accelerando_oie` — Organizational Intelligence Engine

**Role:** "Crystal Reports for activity logs." Telemetry → reasoning → governance → leadership dashboard. Multi-model consensus (QC_MESH 4 models) validates insights before leadership sees them.

**Owns:** Strategic dashboards, bottleneck identification, automation candidate surfacing, trend analysis, anomaly detection.

**Deployment phase:** Months 13–15. **Requires 90+ days of clean post-ERP-go-live telemetry** to have credible baseline.

**Module coverage (from OIE .agi):**
- Daily batch reasoner (overnight: process yesterday's telemetry, surface insights)
- Weekly trend reasoner (Sunday night: trend analysis, week-over-week comparisons)
- On-demand reasoner (user-triggered: deep-dive on specific question)
- Personal coach reasoner (per-leader, derives behavioral suggestions)
- QC_MESH 4-model consensus on every insight before leadership sees it
- NBVE for ongoing model quality governance

**Configuration decisions:**

| Decision | Options | Default |
|---|---|---|
| Telemetry retention | 90 days / 1 year / 3 years | 1 year (audit + trend); summarize older |
| Insight confidence threshold | 0.7 / 0.85 / 0.95 | 0.85 — balance signal vs noise |
| Leadership dashboard refresh | Real-time / daily / weekly | Daily for tactical; weekly for strategic |
| Automation candidate threshold | Frequency × time saved | >50 occurrences/month, >5 min/each |
| Personal coach opt-in | All leaders / opt-in | Opt-in (some leaders find it intrusive) |

**Integration patterns:**
- ERP telemetry → OIE ingestion (primary feed).
- QMS NCR data → OIE quality analytics.
- PI CoE project data → OIE improvement tracking.
- LMS compliance data → OIE workforce readiness.
- Interchange acknowledgement data → OIE customer-relationship health.

**Common dashboards by role:**

| Role | Dashboard view | Update cadence |
|---|---|---|
| Owner / CEO | Revenue, margin, working capital, top 3 risks | Weekly |
| COO | OTD, OEE, throughput, scrap %, top 5 bottlenecks | Daily |
| CFO | DSO, DPO, inventory turns, cash conversion cycle | Daily |
| Quality Mgr | NCR trend, CAPA cycle time, audit readiness | Daily |
| HR | Training compliance %, turnover trend, new-hire ramp | Weekly |

**Gotchas:**
- **The 90-day-clean rule:** Don't trust OIE insights from the first 90 days post-go-live. Data is too messy. Year-1 OIE is for learning the system; year-2 OIE is for trusting the recommendations.
- **The confidence-threshold balance:** Too high → no insights surface; too low → noise. Start at 0.85, adjust based on leadership feedback at month 18.
- **The personal-coach feature:** Some leaders find it valuable, some find it Big-Brother. Opt-in only.

---

## L3 — The anti-pattern catalog (expanded)

All Baby Step anti-patterns A1–A10 are here, plus deeper failure modes from real-world deployments.

### A1. IT-led, not operations-led [BABY]

See Baby Step §A1. The most fatal anti-pattern. If you see only one symptom, see this one.

Additional indicators at scale:
- Steering committee runs at month-end "to fit IT's reporting cycle"
- Vendor selection happened before COO was named sponsor
- IT chose the implementation partner; ops never met them before kickoff
- Configuration decisions made by "best practice from the vendor" rather than fit to the actual operation

**The deeper fix:** The COO's job for 6 months is the deployment. If their day-job won't allow it, defer the project until it can. Half-attention from the operational lead is worse than full-attention from a temporary one.

### A2. Big-bang go-live [BABY]

See Baby Step §A2. The Hershey/FoxMeyer/HP/Lidl pattern.

Additional warning signs:
- Single "cutover weekend" mentioned in the schedule
- Project plan has no module-by-module phasing
- Vendor compensation structure incentivizes faster delivery (% of project on go-live)
- "We don't want to maintain two systems" — true but misframes the choice

**The deeper fix:** Module-by-module with 30-day fallback. The "cost" of dual operations for 60–90 days is real but bounded; the cost of a failed big-bang is unbounded.

### A3. Master data deferred [BABY]

See Baby Step §A3.

Additional patterns:
- "Item naming standard" delayed past month 2
- BOM review treated as engineering-only (not ops + engineering + costing together)
- Customer master deduplication not staffed
- Vendor master cleaned only for active suppliers (the inactive ones still have transactions if you migrate history)

**The deeper fix:** Master data has a named owner (typically Plant Mgr + Engineering Mgr jointly), a deadline (end of month 3), and a quality gate (owner samples 100 items, 95+ correct). Miss the gate, delay Phase 2.

### A4. Customization explosion [BABY]

See Baby Step §A4.

Additional patterns:
- Backlog of customization requests grows weekly
- Engineering changes accumulate during deployment ("while we're at it...")
- Each department defends its current process as sacred
- Vendor quietly encourages customization (more billable hours)

**The deeper fix:** Hard rule: **zero process customization in year 1.** Two written exceptions allowed: (1) customer-mandated EDI variants you can't refuse, (2) regulatory line-items you can't refuse. Everything else: change the process to match the framework. Year-2 retrospect: which customizations would you actually have needed? Usually <20% of what was requested.

### A5. Training as afterthought [BABY]

See Baby Step §A5.

Additional patterns:
- Training week scheduled the week of go-live
- Training delivered only to "power users"
- Training material is the system's UI walkthrough (not workflows)
- Operators told "you'll learn it as you go"

**The deeper fix:** LMS rollout months 7–9. Operators certified on simulators or sandbox environment before any production access. Training is scenario-based ("complete this fictitious work order"), not feature-based ("here are the buttons").

### A6. Year-1 PI CoE [BABY]

See Baby Step §A6.

Additional patterns:
- Eagerness to "show improvement quickly" overrides the data discipline
- Vendor promises "quick wins" — quick wins from chaos aren't sustainable
- Improvement project chartered with no baseline metric
- DMAIC "Define" phase starts before "Measure" data exists

**The deeper fix:** PI CoE starts month 13+. First DMAIC project requires explicit baseline (90 days of post-go-live data, signed off by the COO). Patience here pays for the next decade.

### A7. Custom EDI map proliferation [BABY]

See Baby Step §A7.

Additional patterns:
- "Just this one customer" becomes 40 customers
- EDI mapper turnover means the maps become unsupportable
- Compliance with X12 standards drifts because customers request "their format"
- Mapping debt becomes a soft moat that someone weaponizes against you in due diligence

**The deeper fix:** Standard X12 only. Customers wanting variants either accept standard or pay (in your terms, not theirs). The 1% of customers who'd walk over this aren't worth the systemic debt of accommodating them.

### A8. Forgetting cycle counting [BABY]

See Baby Step §A8.

### A9. Underestimating OEM EDI timelines [BABY]

See Baby Step §A9. The single most-blown manufacturing-ERP deadline is "EDI live with [OEM]." Always.

### A10. No go-live gates [BABY]

See Baby Step §A10.

### A11. Misaligned costing method change

**Symptom:** Existing manufacturer is on actual costing; deployment team enthusiastic about standard costing. Cuts over without proper variance setup.

**Why it fails:** Standard costing requires accurate standards (set annually) and variance accounts (set up correctly). Without them, "standard cost" reports are noise. Margin analysis breaks for 6 months.

**The fix:** If changing costing method, do it as a separate project AFTER ERP is stable (month 18+). Year 1: match existing costing method. Year 2: convert if justified.

### A12. Multi-tenancy assumed but not configured

**Symptom:** Plant 2 onboards; finds out PO numbers collide with Plant 1; quality data mixes across plants in dashboards.

**Why it fails:** Multi-tenancy isn't free even with the framework's support. Document number ranges, role permissions, dashboards-per-tenant, EDI mappings — all need per-tenant config.

**The fix:** Multi-tenancy planned in Config phase even if only one plant initially. Cheap to plan; expensive to retrofit.

### A13. Currency / multi-currency assumed

**Symptom:** Manufacturer with Canadian / Mexican subsidiary goes live; FX gain/loss not configured; intercompany transactions don't balance.

**Why it fails:** Multi-currency adds GL complexity (revaluation, translation, hedging). If not configured properly, monthly close becomes painful.

**The fix:** If multi-currency at go-live: full GL revaluation/translation configuration, hedging treatment defined, intercompany matching active. Plan for this in master data phase.

### A14. Shift-pattern mismatch

**Symptom:** Configured for 2-shift; actual operation runs 3-shift on certain product lines.

**Why it fails:** Time-clock data, OEE calculations, scheduling all assume specific shift patterns. Mismatch causes garbage data.

**The fix:** Survey actual shift patterns BEFORE Config phase. Capture exceptions (specific lines or product families). Config can handle multi-pattern; you have to tell it.

### A15. Forgetting the auditor

**Symptom:** Year-2 audit reveals: not all NCRs have effectiveness checks completed; some training records missing; calibration cycle drifted.

**Why it fails:** "We'll deal with audit when it comes" → it comes and we don't deal well.

**The fix:** Internal audit cycle starts month 9 (with QMS). Quarterly slice audits. Findings become CAPAs immediately. By year-2 external audit, system is genuinely ready.

### A16. The "we don't need cycle counting because we have RFID" trap

**Symptom:** Manufacturer invested in RFID tracking; assumes inventory accuracy is automatic.

**Why it fails:** RFID tracks the tag, not the item. Tags get damaged, missed, applied wrong. RFID without cycle counting often has WORSE accuracy than periodic counting, because nobody verifies.

**The fix:** Cycle counting regardless of RFID. RFID supplements; doesn't replace.

### A17. Migrating bad data

**Symptom:** Decision made to migrate 15 years of transaction history to the new ERP.

**Why it fails:** Old data quality is unknown. Migration takes 3x longer than expected. Migrated data complicates testing. Reports show inexplicable old-data anomalies.

**The fix:** Migrate **balances** + 13 months of comparable history (for prior-year reporting). Retain legacy system as read-only for older transactional history. Saves 60-80% of migration effort.

### A18. Ignoring change management

**Symptom:** Project is technically excellent; on time, on budget, all features working. Adoption is 40% three months post-go-live.

**Why it fails:** The system isn't the system; the workflows are the system. People's daily work changed but their incentives, recognition, and habits didn't. Reversion is rational.

**The fix:** Change management is 30–50% of the project. Dedicated change-management lead (sometimes external coach). Communication plan. Wins celebrated publicly. Resistance addressed individually. Manager 1:1s in months 1–3 post go-live to surface struggles.

### A19. Underestimating month-end close in new system

**Symptom:** First month-end after go-live takes 10 days instead of 3.

**Why it fails:** Accountants navigating an unfamiliar system, reconciling new data to old patterns, identifying conversion-era anomalies.

**The fix:** Month-end rehearsal in UAT (month 4–5). Detailed runbook for first 3 closes. Schedule double the normal close time for first 3 months; reset expectations as team adapts.

### A20. Year-1 reporting requirements unknown

**Symptom:** Manufacturer goes live; finance team asks for reports they had in legacy system; reports don't exist.

**Why it fails:** Reporting requirements never fully captured during requirements; legacy reports often built ad-hoc by users.

**The fix:** Report inventory in master data phase. For each legacy report: is it actually used (monthly? quarterly? at all?)? What decisions does it inform? Replace high-use reports first; sunset never-used ones.

### Risk → mitigation extended matrix

| Risk pattern | Likelihood | Impact | Mitigation | Detection |
|---|---|---|---|---|
| Master data quality <95% | High | Existential | Phase 1 cleansing; owner gate | Sample audit at month 3 |
| Operator training skipped | Medium | High | LMS rollout months 7–9; cert gate | LMS dashboard at month 6 |
| OEM EDI integration delays | High | Medium | Manual fallback; submit month 1 | OEM acknowledgment of test slot |
| Customization backlog explodes | High | Medium | No-customization year-1 rule | Backlog count >10 = red flag |
| Executive sponsor disengages | Medium | Existential | Monthly QBR owner attendance required | Missed two consecutive QBRs |
| Cycle counting deferred | High | Medium | Daily ABC from day 1; accuracy gate | Cycle count completion % |
| PI CoE started in year 1 | Medium | Low | 90-day rule, PI CoE month 13+ | Project charter dates |
| Big-bang attempted | Low at mid-size | Existential | Contractually phased | Schedule reviewed at kickoff |
| Costing change mid-rollout | Low | High | Defer to year 2 | Costing decision in Config intake |
| Multi-currency uncovered late | Medium | Medium | Currency assessment in Config | Subsidiary list at month 1 |
| Auditor surprise at year 2 | High if ignored | Medium | Quarterly slice audits from month 9 | CAPA cycle time, training % |
| Reporting requirements gaps | High | Low | Report inventory in master data phase | Month-end close pain |
| Adoption <80% at 90 days | High if change mgmt skipped | High | Dedicated change-mgmt lead | Eliza usage % at 90 days |

---

## L4 — Scenarios (5 archetypes)

Every scenario below is a complete deployment narrative — situated problem → recommended composition → sequencing → first-90-days plan → key risks → success criteria. Use these as direct templates; substitute the customer's specifics.

### Scenario A — Greenfield discrete manufacturer

**Background:** Acme Machining, 200 employees, $80M revenue. Contract CNC machining, 75% automotive Tier-2 (Ford, GM, Stellantis suppliers), 25% aerospace Tier-2. Currently: QuickBooks, three Excel scheduling workbooks, paper travelers on shop floor, 1990s desktop quality system. Owner is 58, succession planning for sale in 5–7 years.

**Goal:** Full Accelerando Enterprise Core stack live in 18 months. ISO 9001 audit-ready by month 18. IATF 16949 audit-ready by month 24.

**See Baby Step §L4 — Phase 1–6 plan in detail.** Super-doc additions follow.

#### Year-2 plan (post-deployment)

- **Q1 (Months 19–21):** IATF 16949 certification audit. Sustained ≥95% LMS compliance, sustained ≥98% inventory accuracy, all year-1 CAPAs closed with effectiveness checks. Estimated audit prep: 4 weeks of Quality Mgr time.
- **Q2 (Months 22–24):** AS9100 readiness gap analysis for aerospace customer growth. Estimated: 6 months from start to audit-ready (AS9100 is more restrictive than IATF).
- **Q3 (Months 25–27):** Customization debt review. Year-1 accumulated customizations cataloged; eliminate 50%+ by reverting to standard configurations.
- **Q4 (Months 28–30):** OIE-driven bottleneck project. Top constraint identified from year-2 telemetry; PI CoE chartered with 6-month improvement target.

#### KPI targets (year 1 vs year 2)

| Metric | Baseline | Year-1 target | Year-2 target |
|---|---|---|---|
| On-time delivery | 87% | 94% | 97% |
| First-pass yield | 91% | 94% | 96% |
| Inventory turns | 6.2 | 8.5 | 10.0 |
| DSO | 52 days | 42 days | 35 days |
| CAPA cycle time | n/a | 45 days avg | 30 days avg |
| Training compliance % | n/a | 90% | 95% |
| Top-5 NCR cat reduction | n/a | -25% | -50% |
| OEE | n/a | 60% measured | 70% |

#### Year-1 budget envelope (typical)

- Software licenses (Accelerando, infrastructure): $80–120k/year for this size
- Implementation services (internal + external): $400–600k one-time
- Hardware (Eliza tablets, kiosks, server): $40–80k one-time
- Training delivery (LMS content, instructor time): $50–100k one-time
- Change management coaching: $30–60k one-time
- Contingency (15%): $90–135k
- **Total year-1: $690k–$1,095k. Typical landing point: ~$850k for $80M-revenue manufacturer.**

ROI typically realized in months 18–30:
- DSO improvement: ~$1.5M working capital release
- Inventory turn improvement: ~$1.2M working capital release
- Scrap reduction (3% of revenue typical): ~$2.4M/year ongoing
- Productivity (less expediting, fewer chargebacks): ~$800k/year ongoing

#### Critical risks for this scenario

1. **Owner succession pressure.** If sale is planned in 5 years, audit-ready financials and operational systems become major value drivers. Don't compromise on quality of master data — this becomes due-diligence material.
2. **Automotive customer concentration.** 75% to three OEMs means EDI integration is existential. Submit all three OEM EDI requests in month 1; have manual fallback for whichever OEM IT lags.
3. **Aerospace customer pull.** If aerospace customer base grows during year-2, AS9100 timing pulls in. Plan for early IATF cert (Q1) leaving Q2-Q4 for AS9100 work.

---

### Scenario B — Legacy ERP replacement

**Background:** Precision Stamping & Fabrication, 280 employees, $95M revenue. Fabricator + stamper for industrial OEMs (Caterpillar, John Deere, Bobcat). Running on Macola Progression (acquired by Exact Software 2010, end-of-life). Current ERP has 15 years of data, multiple home-grown reports, 30+ customizations. Owner ready to invest in replacement; CFO nervous about the migration; COO supportive but burned by an earlier failed Microsoft Dynamics attempt 8 years ago.

**Goal:** Full Accelerando deployment in 20 months. No production downtime. Legacy data preserved per 7-year retention policy.

#### Sequencing variations from default arc

The big difference from Scenario A is **Phase 1 extends to 4 months** (data extraction + cleansing from legacy is heavier).

| Phase | Months | Activities |
|---|---|---|
| 1 — Discovery, Config, master data extraction | 1–4 | Config intake; legacy data extraction in parallel; deduplication; engineering BOM review |
| 2 — ERP UAT and parallel run | 5–8 | UAT with extracted data; parallel run for 60 days with both systems live |
| 3 — Cutover and stabilization | 9–10 | Cutover weekend; 30-day intensive stabilization; legacy frozen read-only |
| 4 — Quality + training | 11–13 | QMS + LMS deployment |
| 5 — Integration + operator UI | 14–16 | Interchange + Eliza + Chatbot |
| 6 — Improvement + intelligence + governance | 17–20 | PI CoE + OIE + Legal + ES |

#### Legacy data migration playbook

**Months 1–2 — extraction:**
- Master data: items, BOMs, routings, customers, vendors, GL chart → extracted with quality flags
- Open transactions: open SOs, open POs, in-process WOs → extracted with full detail
- Balances: AR aged, AP aged, GL trial balance → extracted at cutover date
- History: 13 months of transactions (for prior-year comparison) → extracted summarized

**Months 2–3 — cleansing:**
- Item master deduplication (typically 20-40% duplicates at 15-year shop)
- BOM engineering review (phantoms, obsolete revisions, missing operations)
- Customer master deduplication (slight name variants from years of data entry)
- Vendor master deduplication
- GL chart review (likely fine, sometimes needs restructuring)

**Months 3–4 — validation:**
- Sample 100 items per master category; 95+ must be correct
- Owner sign-off per category
- Reconciliation: trial balance from legacy = trial balance in new system at cutover date
- Open transaction reconciliation: every open SO/PO/WO accounted for

**Month 5 onward — UAT:**
- Three full end-to-end cycles with real (cleansed) data
- Reporting validation: 20 most-used legacy reports replicated in new system
- Month-end close rehearsal

#### Cutover weekend playbook

Friday 5pm: Legacy system locked. Final balances captured.
Friday 6pm – Saturday 6am: Balance migration runs. Trial balance reconciles.
Saturday 6am – Sunday noon: Configuration final-fixes; sample transactions tested.
Sunday noon: Go/no-go decision with COO + IT.
Sunday afternoon: Operators trained on go-live procedures.
Monday 7am: New system live. Stabilization team in place. Legacy in read-only mode.

#### Critical risks for this scenario

1. **The "failed Dynamics" memory.** The COO's previous bad experience colors everything. Acknowledge it explicitly in kickoff. Address differences: phased not big-bang, ops-led not IT-led, no customization year-1. Build trust through small early wins.
2. **30+ legacy customizations.** Most are unnecessary or obsolete. Pre-cutover review: which are still used (operational telemetry from legacy proves it)? Which are mandated by customers? Migrate only what's required.
3. **Multiple home-grown reports.** Power users in finance built reports over years. Migration won't move these automatically. Plan: inventory them, prioritize by use frequency, rebuild top 10 in new system, sunset rest.
4. **CFO nervousness.** Provide CFO with weekly migration dashboard (data quality % migrated, reconciliation status, gap list). Visibility reduces anxiety; opacity increases it.

#### Year-1 budget addition for migration

Add to base budget:
- Migration extraction & cleansing services: $150–250k
- Parallel run cost (operating two systems): $50–80k
- Stabilization staffing (consultants on-site for 30 days): $60–100k
- Total migration premium: **+$260k to $430k over greenfield budget.**

---

### Scenario C — Multi-plant rollout

**Background:** Apex Industries, 4 plants (Indiana 180 employees, Tennessee 120, Texas 95, Mexico 75 — total 470 employees, $145M revenue). Discrete manufacturing — industrial equipment subassemblies for OEMs. Each plant currently runs different systems: Indiana on legacy Epicor 9 (2015), Tennessee on QuickBooks + Excel, Texas on Accelerando already (deployed last year — they're the pilot), Mexico on a Mexican ERP (Intelisis) with Spanish-language operators.

**Goal:** All 4 plants on a single Accelerando instance within 24 months from start. Consolidated financials. Cross-plant inventory visibility. Shared customer master.

#### Sequencing strategy

Texas is already on Accelerando — they're the pattern + the cautionary tales. The other 3 plants roll out sequentially, NOT in parallel. Each plant takes 6–9 months, with overlap.

| Plant | Months | Notes |
|---|---|---|
| Texas | Month 0 (already done) | Pattern source |
| Indiana | Months 1–9 | Largest plant; biggest legacy challenge (Epicor migration) |
| Tennessee | Months 7–14 | Smallest; easiest (greenfield-ish from QuickBooks) |
| Mexico | Months 13–22 | Spanish-language; FTA/CUSMA compliance; cultural change-management focus |

Total: 22 months for full rollout (24-month goal achievable).

#### Multi-plant configuration decisions

| Decision | Options | Recommendation |
|---|---|---|
| Tenancy model | Separate tenants per plant / one tenant multi-org | One tenant with org dimension — shared customer/vendor master, plant-specific inventory/manufacturing |
| Number ranges | Per-plant prefixes (IN-SO-001, TN-SO-001, TX-SO-001, MX-SO-001) | Yes — operationally clearer |
| Inventory transfer | Manual / automatic | Inter-plant transfer workflow active |
| Consolidated AR | Centralized AR / per-plant | Centralized AR; cash app at HQ |
| Consolidated AP | Centralized AP / per-plant | Centralized AP; PO origination per plant |
| GL consolidation | Per-plant + consolidated entity | Per-plant ledgers; consolidation entity for management reporting |
| Multi-currency | USD for US, MXN for Mexico, USD reporting | Yes — both transaction and reporting currency configured |
| Language | English / Spanish (Mexico) | Bilingual UI for Mexico plant; Spanish documentation |
| EDI mapping | Per-plant customer relationships | Per-plant if customer relationship is per-plant; consolidated if customer ships to multiple plants |

#### Plant-by-plant playbook

**Texas (already done):** Document everything. Texas is the source of patterns. Every config decision Texas made should be available to subsequent plants. Texas's CAPAs are signal of what other plants will hit.

**Indiana — months 1–9:**
- Phase 1 (1–4): Legacy Epicor extraction + cleansing
- Phase 2 (5–6): UAT + cutover, leveraging Texas patterns
- Phase 3 (7–9): QMS + LMS + integration with Texas (inter-plant transfer, consolidated reporting)

**Tennessee — months 7–14:**
- Phase 1 (7–9): Greenfield-ish from QuickBooks + Excel; faster than Indiana
- Phase 2 (10–11): UAT + cutover
- Phase 3 (12–14): QMS + LMS + integration

**Mexico — months 13–22:**
- Phase 1 (13–16): Spanish-language setup; FTA/CUSMA compliance review; cultural change-mgmt prep
- Phase 2 (17–18): UAT + cutover with bilingual support
- Phase 3 (19–22): QMS + LMS in Spanish; integration; export compliance

#### Critical risks for this scenario

1. **The "make Indiana like Texas" trap.** Indiana operators will resist patterns from Texas if they perceive them as "imposed." Frame: "We're learning from Texas; here's where Indiana's reality is different and the config will reflect that." Genuine differences honored.
2. **Mexico language + culture.** Spanish-language UI is necessary but not sufficient. Local cultural norms around hierarchy, training, and change differ. Have a Mexico-based change-mgmt lead, not a remote one.
3. **CUSMA / FTA compliance.** Cross-border manufacturing has duty/tariff complexity. Country-of-origin tracking, certificate-of-origin generation, harmonized tariff classification. Configure carefully.
4. **Master data harmonization.** Indiana, Tennessee, Texas may have the same item under different part numbers. Mexico almost certainly does. Harmonization is a year-2+ project, NOT a Day-1 project (per A18-equivalent rule).
5. **Consolidated reporting timing.** Each plant's monthly close happens locally; consolidation runs at corporate. First few consolidations will reveal data inconsistencies. Plan for slower consolidation in months 1–6 post-final-plant.

---

### Scenario D — M&A plant integration

**Background:** TitanWorks (already running Accelerando, 18 months in, $120M revenue) acquired Specialty Gears Inc — 75 employees, $20M revenue, specialty gear manufacturer for aerospace and industrial. Specialty Gears runs on an old MAPICS MRP system (IBM mainframe, deployed 1995) and three spreadsheets. Acquisition closed; integration must happen within 6 months for accounting consolidation.

**Goal:** Specialty Gears fully operating on TitanWorks' Accelerando instance within 6 months. AS9100 certification (which Specialty Gears holds independently) preserved through transition.

#### Compressed 6-month playbook

| Month | Activities |
|---|---|
| 1 | Discovery + integration planning; data extraction begins; cultural assessment |
| 2 | Master data cleansing; configuration of Specialty Gears as new tenant in existing instance |
| 3 | UAT with Specialty Gears master data; first parallel run cycle |
| 4 | Cutover; legacy MAPICS frozen; 30-day stabilization |
| 5 | LMS rollout for Specialty Gears operators on TitanWorks procedures; QMS continuity validation |
| 6 | Integration touchpoints (inter-plant transfers if applicable, consolidated reporting, EDI for shared customers) |

#### M&A-specific decisions

| Decision | Recommendation |
|---|---|
| Tenancy | New tenant in existing Accelerando instance, NOT new instance |
| AS9100 | Maintain Specialty Gears certification separately for 12 months; merge audit cycle in year-2 if audit body permits |
| Part numbers | Preserve Specialty Gears part numbers for 12 months (customers know them); harmonization year-2+ project |
| Customer master | Audit for overlap with TitanWorks customers; consolidate AR by customer where appropriate |
| Vendor master | Audit for overlap; consolidate purchasing power where customer-permission allows |
| GL | Extend TitanWorks chart of accounts to include Specialty Gears; don't import their chart |
| EDI | Specialty Gears' existing customer EDI mappings: preserve where customer relationship continues; sunset where TitanWorks already has the relationship |
| Operators | All Specialty Gears operators need 90-day LMS catch-up on TitanWorks procedures |
| Quality history | Migrate 24 months of CAPA history for AS9100 audit continuity |

#### Cultural integration playbook

M&A integrations fail more often on culture than systems. Specific tactics:

1. **Specialty Gears retains its name and identity for 12 months minimum.** Branding change is a separate decision, not bundled with system change.
2. **Specialty Gears' COO becomes a peer to TitanWorks' COO** (not a subordinate). Both report to the parent COO. Equal voice in the joint steering committee.
3. **Communication cadence:** Weekly all-hands at Specialty Gears for first 90 days. Acquisition leadership visible.
4. **Quick wins for Specialty Gears people:** Identify 2–3 things their old MAPICS couldn't do that Accelerando does. Demonstrate. Win trust.
5. **Job security messaging:** No layoffs from system migration. If consolidation requires reduction, separate decision, separate timeline, after the migration is complete.

#### Critical risks for this scenario

1. **Acceleration pressure from accounting.** Accounting wants consolidated financials yesterday. The 6-month timeline is aggressive; resist further compression. Failure here is operationally catastrophic.
2. **Loss of key Specialty Gears people.** Acquisitions trigger talent loss. Identify the 3–5 people critical to operational continuity; retention packages.
3. **AS9100 audit timing.** If Specialty Gears' AS9100 audit falls within the 6 months, push to delay or scope to only-the-quality-system (not the new ERP). Mid-migration audit is high-risk.
4. **Cultural rejection of "the corporate system."** Specialty Gears operators may experience Accelerando as imposed. Address directly: show them the audit trail benefits, the rework time saved, the OIE dashboards.
5. **Inter-plant transfer accuracy.** If TitanWorks and Specialty Gears share customers or transfer material, the workflow must be airtight from Day 1 of cutover.

---

### Scenario E — Automotive Tier-2 contract manufacturer with severe customer pressure

**Background:** Midwest Metals, 250 employees, $90M revenue. Tier-2 stamping and welding for Big-3 automotive. Recent year has been brutal:
- Top customer (Ford supplier) issued formal supplier corrective action for delivery (multiple late shipments)
- IATF 16949 surveillance audit (Q1) yielded 6 minor and 2 major findings
- One customer (GM supplier) issued a chargeback for label compliance issues ($45k Q1)
- COO is exhausted; threatening to retire
- Owner is considering whether to invest in systems or sell

Goal: Accelerando deployment as the rescue. 18 months to clean operations, IATF re-cert pass, customer scorecard recovery, and (the owner's silent goal) make the company sale-attractive.

#### Sequencing variations for the rescue context

This is a turnaround scenario. Speed matters. But the discipline of phased rollout matters more — a botched rescue is final.

| Phase | Months | Activities | Customer-pressure response |
|---|---|---|---|
| 0 | Pre-month-1 | Owner + Quality + COO summit. Make the rescue commitment public to customers. Get OEM agreement to defer audit until month 18. | Set expectations: visible improvement plan, no quick fixes. |
| 1 | 1–4 | Config + master data + QMS basics (NCR + CAPA stand-up FIRST, before ERP) | First weekly CAPA progress report to top customer. |
| 2 | 5–8 | ERP core go-live | First month of clean transactional data. |
| 3 | 9–11 | QMS depth + LMS + Interchange | All customer EDI brought into compliance. Label workflow embedded. |
| 4 | 12–14 | Eliza on shop floor + Chatbot for customer self-service | OEM customer scorecards visible improvement. |
| 5 | 15–17 | PI CoE + OIE | First DMAIC closed with measured customer-relevant improvement. |
| 6 | 18 | IATF 16949 re-cert audit | Pass. |

#### Customer-pressure tactics

1. **Transparent CAPA reporting to top customers monthly.** Don't wait for them to ask. "Here's what we did this month on the corrective action you filed." Visible discipline rebuilds trust faster than perfection.
2. **Quality Manager directly accountable to top customer's SQE (Supplier Quality Engineer).** Not through sales. SQE-to-Quality-Mgr relationship is the rebuild axis.
3. **Label compliance as a first-90-days project.** Specific, visible, customer-mandated. Quick demonstrable win.
4. **OEM customer site visit at month 6.** Show them the new shop floor, the new quality system. Trust is visual.

#### COO retention play

The COO is exhausted. Without them, deployment fails. Tactics:
- Bring in an interim Deployment Manager (external) to take operational burden off COO for the 18-month project
- Offer COO a stay bonus tied to successful audit pass at month 18
- Acknowledge their experience explicitly: their pattern recognition is irreplaceable; the new system documents what they know

#### Year-1 business risks

1. **Customer loss during the rebuild.** A top customer might lose patience and re-source. Mitigation: transparent monthly progress; visible early wins.
2. **Cash flow during the rebuild.** Chargebacks reduce AR; investment increases AP. Owner needs to ensure 12 months of working capital available.
3. **Talent loss during stress.** People leave under pressure. Mitigation: visible plan + visible progress = retention.
4. **The "investment instead of sale" question.** Owner may decide mid-deployment to sell anyway. Build the deployment to maximize value-of-the-business regardless: clean financials, clean operations, audit-ready quality, transferable systems. The deployment IS the value creation.

#### Success metrics for the rescue

| Metric | Baseline (Month 0) | Month 6 | Month 12 | Month 18 |
|---|---|---|---|---|
| OTD | 78% | 88% | 93% | 96% |
| Customer chargebacks ($) | $180k/yr run rate | $80k | $30k | $10k |
| IATF surveillance findings | 6 minor + 2 major | n/a (not audited mid-cycle) | 2 minor (internal audit) | 0 findings (re-cert) |
| Top customer scorecard | "At risk" | "Improving" | "Approved" | "Preferred" |
| COO retention | Threatening retirement | Engaged | Recommitted | Owns successor plan |

---

## L5 — Edge cases, regulatory adjacencies, known limits

### Process manufacturing (chemicals, food, ingredients)

Different ontology. The Accelerando ERP module covers batches and lots, but discrete-manufacturing-centric workflows (BOMs, routings, work orders) need to be reframed:

- Batches not work orders
- Recipes not BOMs (with phantom percentage yields)
- Lots not serial numbers (but with deeper traceability requirements — forward and back trace from any lot)
- Expiration dates everywhere
- Tank-and-line equipment vs discrete work centers
- FIFO mandatory (for shelf-life products)
- HACCP (Hazard Analysis Critical Control Points) for food

If a process manufacturer asks: Accelerando can serve them, but the deployment plan is different enough that a separate skill doc is appropriate. Out of v1.0 scope for this doc.

### Regulated industries needing certification

**Out of v1.0 scope for the manufacturing playbook:**
- Pharmaceuticals (FDA 21 CFR Part 11, ICH Q10, GxP) — requires validated systems
- Medical devices (ISO 13485, FDA 21 CFR Part 820) — requires design controls
- Defense (ITAR, DFARS) — requires controlled infrastructure
- Aerospace prime (vs Tier-2) — additional security/quality requirements
- Nuclear (10 CFR 50 Appendix B) — specialized quality framework

These industries can use the Accelerando framework but the deployment playbook is materially different and requires industry-specific expertise. Don't try to wing them.

### Tier-2 vs Tier-1 manufacturer

The playbook is for Tier-2 (suppliers to Tier-1, who supply to OEMs). Tier-1 has:
- Larger scale (typically 1,000+ employees)
- More complex EDI surface (multiple OEM relationships, sub-tier supplier portals)
- Tooling and engineering responsibility (own product development)
- Capital equipment investments at OEM plants
- Different audit regime (OEM-specific quality systems beyond IATF)

If a Tier-1 asks: many patterns transfer but the scale shift requires a different playbook (closer to enterprise scale).

### Multi-currency and international manufacturing

Briefly covered in Scenario C (Mexico). Detailed multi-currency considerations:

- Transaction currency vs functional currency vs reporting currency
- FX revaluation accounting (gain/loss on monetary items)
- Translation methodology (current rate vs temporal)
- Hedging treatment (cash flow hedges, fair value hedges)
- Intercompany matching and elimination
- Transfer pricing documentation (BEPS / OECD)
- Withholding tax considerations
- VAT/GST (non-US)

Get a CFO + tax advisor involved in Config phase if multi-currency. Don't wing it.

### Capital equipment manufacturers (long-cycle, large units)

Single units, 6–18 month build cycles, customer-specific engineering, milestone billing. ERP configuration:
- Project module heavy (each unit IS a project)
- Engineering BOM separate from manufacturing BOM
- Progress billing per milestone (not on ship)
- Field service module active (long warranty, support)
- Customer asset tracking (the OEM customer owns equipment in your plant during build)

### Contract manufacturers vs OEMs

| Dimension | Contract manufacturer | OEM (own product) |
|---|---|---|
| Engineering | Customer-provided drawings | Internal engineering |
| Engineering changes | ECN from customer | ECO internal |
| Inventory ownership | Customer-owned consigned often | All owned |
| Tooling | Customer-owned tooling on your floor | Owned |
| Pricing | Negotiated per program | Catalog + customer terms |
| Quality | Customer-mandated (IATF, AS9100, OEM-specific) | Internal + industry standard |
| EDI footprint | Heavy (every customer integration) | Moderate (sales channels) |
| Sales cycle | Long (program awards) | Variable |

Configuration shifts: contract manufacturers configure CRM lightly, ERP project module heavily. OEMs vice versa.

### Engineering Change Order (ECO) discipline

Often skipped, often regretted. The ECO workflow:

1. Engineer proposes change
2. Cross-functional review (engineering, manufacturing, quality, costing)
3. Customer notification (mandatory for customer-controlled designs)
4. Approval
5. Effective date set
6. BOM/routing updates
7. Existing inventory disposition (use up, scrap, rework)
8. Training updates (if work instructions change)
9. Communication to shop floor
10. Confirmation of execution

ERP + QMS + LMS together support this. Configure ECO workflow in QMS, integrate with ERP BOM update, trigger LMS re-cert if procedural change.

### The "we already have ISO 9001, why do we need a QMS module" trap

Many manufacturers have ISO 9001 certification but the system is document-management-with-audit-anxiety. Real QMS = closed loop. The conversation:

- "How long is your average CAPA cycle?" If they don't know: red flag.
- "Show me your last management review meeting minutes." If hard to find: red flag.
- "What's the trend on top NCR category?" If they answer with anecdote not data: red flag.
- "How many audit findings from last external audit?" If they don't track findings to closure: red flag.

The QMS module isn't an addition to their ISO; it's the implementation of what ISO actually requires.

### Multi-language operator considerations

Discrete manufacturing in the US often has Spanish-speaking operators. Considerations:

- Eliza UI in Spanish (the framework supports localization at codegen time)
- LMS content in Spanish (translate; verify with native-speaker operators)
- QMS NCR initiation in Spanish (operator describes the problem in their language)
- Work instructions in both English and Spanish (visible at the work cell)
- Cultural adaptations: hierarchy norms, training approach, change management

Other languages (French in Quebec, Portuguese in some operations) follow same pattern.

### Year-2 onward: when does the deployment ever end?

Deployments don't end at month 18. They transition to operations + continuous improvement:

- **Year 2:** Customization debt reduction, second-wave training, OIE recommendations operationalized, PI CoE running 4–8 projects per year
- **Year 3:** New module additions if business expands (additional plants, new product lines, M&A)
- **Year 4–5:** Major version upgrade of Accelerando, refactoring of any accumulated debt
- **Year 5+:** The system is now organizational infrastructure; treat it like the building's electrical system — invest in maintenance, plan for periodic upgrades

The skill doc you're reading covers months 1–18. Year-2+ is a separate playbook (and a future skill doc).

---

## L6 — Rubric self-check prompts

Each prompt below is a scenario the AI advisor should be able to handle. The expected shape is a numbered rubric — each item is present (pass) or absent (fail). The rubric tests domain knowledge, structural completeness, and pragmatic sequencing.

### Self-check 1 — Greenfield discrete manufacturer (deep)

**Prompt:** Contract machining shop. 180 employees. $65M revenue. Customer mix: 60% automotive Tier-2, 30% industrial OEM, 10% aerospace Tier-2. Currently: a legacy Sage 100 from 2012, Excel scheduling, paper travelers. Owner is 55, planning to sell to son-in-law in 7 years. Walk through the full 18-month deployment plan.

**Expected rubric (each item is pass/fail):**
1. Deployment sequence: Config → Master Data → ERP core → Billing → QMS → LMS → Interchange → Eliza → Chatbot → PI CoE → OIE → Legal → ES
2. Master data Phase 1 (months 2–3) with owner sign-off gate (sample 100 items, 95+ correct)
3. Cycle counting from day 1 of ERP go-live; ABC classification; ≥98% accuracy as go-live gate
4. Training (LMS) rolled out months 7–9, BEFORE ERP go-live for affected systems; ≥95% cert before login
5. Identifies **IATF 16949** as automotive Tier-2 compliance target (not AS9100), with year-2 cert audit
6. Identifies **AS9100** as aerospace tier compliance target, with deferred year-2-or-3 cert
7. EDI integration timeline acknowledges OEM IT scheduling: requests submitted month 1, live month 10–12
8. Executive sponsor: COO or owner (NOT IT Director)
9. Phased go-live (never big-bang) with 30-day fallback per module
10. PI CoE starts month 13+, never earlier, with 90-day-clean-telemetry rule
11. Go-live gates documented: master data sign-off, training completion %, UAT cycles, rollback plan
12. KPIs named with baseline + targets: OTD, first-pass yield, inventory turns, DSO, CAPA cycle time
13. Year-2 plan includes IATF certification audit, customization debt reduction, AS9100 readiness
14. Budget envelope estimated ($700k–$1.1M for this revenue size); ROI timeline (18–30 months)
15. Critical risks named: master data quality, customer concentration, EDI timeline, change management
16. Succession-planning context honored: deployment increases sale-attractiveness (year-7 owner exit)

### Self-check 2 — Legacy ERP replacement (Macola/MAPICS/Sage 100)

**Prompt:** Manufacturer running Macola Progression for 12 years; ready to replace. 240 employees, $85M revenue, industrial OEM customer base. Owner is conservative (burned by a failed Microsoft Dynamics rollout 6 years ago). CFO wants risk minimization above all else.

**Expected rubric:**
1. Phase 1 extended to 4 months (legacy extraction + cleansing is heavier than greenfield)
2. Total timeline 18–22 months (slower than greenfield)
3. Parallel-run period explicitly planned (60–90 days dual operation, not flip-the-switch)
4. Master data extraction strategy: items, BOMs, routings, customer, vendor, GL chart (months 1–2)
5. Master data cleansing: dedup (20–40% duplicates typical at 12-year shop), engineering BOM review (months 2–3)
6. Open transaction handling: open SOs/POs/WOs migrated with explicit accounting
7. Balance migration approach: trial balance at cutover date; 13 months history for prior-year reporting; older history read-only in legacy
8. Cutover-weekend playbook documented (Friday lock → Sunday go/no-go → Monday live)
9. 30-day stabilization plan with on-site consultants
10. Legacy system frozen read-only for 7-year retention
11. Reporting migration: legacy report inventory, top 10 rebuilt, rest sunset
12. Risk acknowledgment of "failed Dynamics" memory: direct address in kickoff; differences spelled out
13. CFO weekly migration dashboard (data quality %, reconciliation status, gap list) — visibility for risk-anxious CFO
14. Customizations review: most legacy customizations obsolete; sunset before migration
15. Migration premium budgeted: +$260k–$430k over greenfield baseline
16. Stabilization staffing plan (consultants on-site during first 30 days post-cutover)

### Self-check 3 — Multi-plant rollout (4 plants, 2 years)

**Prompt:** Industrial subassembly manufacturer. 4 plants: Ohio 220 employees, Michigan 165, Arizona 110, Tijuana Mexico 90. Total 585 employees, $175M revenue. Each plant on different systems. Owner wants consolidated financials, cross-plant inventory visibility, shared customer master, in 24 months.

**Expected rubric:**
1. Plants rolled out SEQUENTIALLY, not in parallel
2. Pilot plant identified first (smallest or simplest legacy); used as pattern source
3. Tenancy model: one tenant with org dimension (not separate tenants), with rationale
4. Number ranges per-plant prefixed (OH-, MI-, AZ-, MX-) for operational clarity
5. Inter-plant transfer workflow active from plant 2 onward
6. Consolidated AR/AP centralized at HQ
7. GL: per-plant ledgers + consolidation entity for management reporting
8. Multi-currency for Mexico: USD/MXN, both transaction and reporting; FX revaluation configured
9. Spanish-language UI + LMS content for Tijuana
10. Mexico-based change management lead (NOT remote)
11. CUSMA/FTA compliance: country-of-origin tracking, certificate-of-origin generation
12. Plant sequencing rationale: Ohio first (largest + legacy challenge), then mid, then Mexico last (most complex change)
13. Total timeline ~22 months for 4-plant rollout
14. Master data harmonization (cross-plant part number unification) deferred to year 2+, NOT Day 1
15. Cultural risk: "make Indiana like Texas" trap acknowledged with mitigation
16. Consolidated reporting timing expectation: slower for first 6 months post-final-plant

### Self-check 4 — M&A integration (6 months)

**Prompt:** Parent company runs Accelerando (18 months in, $130M revenue) just acquired a specialty grinder shop (85 employees, $22M revenue) running MAPICS from 1995. AS9100 certified independently. Need integration in 6 months for accounting consolidation.

**Expected rubric:**
1. Compressed timeline (6 months) acknowledged as aggressive but feasible
2. Six-phase plan: discovery → master data → UAT → cutover → training → integration
3. New tenant in EXISTING Accelerando instance (not new instance)
4. Acquired shop retains its name + identity for 12 months minimum
5. Acquired shop's COO becomes peer (not subordinate) to parent COO
6. Part numbers preserved for 12 months (harmonization deferred to year 2+)
7. AS9100 maintained separately for 12 months; audit cycle merge in year 2 if audit body permits
8. Acquired operators get 90-day LMS catch-up on parent's procedures
9. Customer overlap analysis (AR consolidation where appropriate)
10. Vendor overlap analysis (purchasing consolidation where customer-permission allows)
11. EDI mapping decisions: preserve where customer relationship continues, sunset where parent already has the relationship
12. Quality history migrated (24 months CAPA history for AS9100 audit continuity)
13. Cultural integration tactics: weekly all-hands, visible leadership, quick wins specific to acquired team
14. Retention: identify and retention-package 3–5 critical Specialty Gears people
15. Risk: defer go-live rather than rush if acquired master data is in worse shape than expected
16. AS9100 audit during the 6 months: push to delay or scope down

### Self-check 5 — Anti-pattern detection (deep)

**Prompt:** Critique this proposal:

> "We're a 175-employee CNC shop. Our IT Director will sponsor the project (he's our most technical leader). Vendor will lead implementation. We're going live January 1st across all modules — ERP, QMS, LMS, Eliza, PI CoE, OIE — in one weekend cutover. We'll train operators the week of Christmas. Master data cleanup will happen in months 4–6 after we're live (system will be cleaner anyway). First DMAIC project starts month 3 to demonstrate quick wins. Our 45 customer-specific EDI maps from the current system will all be recreated. We're keeping all 30+ customizations from our existing system. We've never done a parallel run; sounds expensive."

**Expected critique rubric — identifies all of the following anti-patterns:**
- **A1** (IT-led: "IT Director will sponsor" — must be COO or owner)
- **A2** (big-bang go-live: "all modules in one weekend cutover")
- **A2 specific:** January 1st timing is risky for many manufacturers (year-end inventory close, customer ordering patterns)
- **A3** (master data deferred: "happen in months 4–6 after we're live")
- **A5** (training week of Christmas: operators won't be present, won't retain)
- **A6** (PI CoE in month 3: no baseline data, "quick wins from chaos")
- **A7** (45 customer-specific EDI maps recreated: maintenance debt forever)
- **A4** (30+ customizations preserved: customization explosion)
- **A2/A10** (no parallel run: dangerous for legacy migration; not actually expensive vs cost of failure)
- **A10** (no go-live gates mentioned)
- (Bonus) Vendor leading implementation = warning sign for accountability/sponsorship clarity

And rewrites the plan correcting each, with the corrected version using the default 18-month arc, IT in support role, COO as sponsor, phased rollout, master data Phase 1, parallel run, standard EDI only, no customizations year-1, training months 7–9 (with cert gates), PI CoE month 13+, and explicit go-live gates.

### Self-check 6 — KPI conversation with CFO

**Prompt:** CFO of a 240-employee discrete manufacturer asks: "We're considering a $1.5M Accelerando deployment. How will I know it's working? What numbers should I be watching, and what's a realistic timeline for seeing them improve?"

**Expected rubric:**
1. KPIs grouped by CFO-relevance (cash conversion, working capital, margin, operational health)
2. Cash conversion metrics: DSO target reduction (baseline → target), inventory turns improvement
3. Working capital metrics: DPO, cash conversion cycle, working capital release ($ estimate)
4. Margin metrics: first-pass yield → scrap cost reduction (3% of revenue typical), OTD → expediting cost reduction
5. Operational health metrics: top-5 NCR trend, CAPA cycle time, training compliance %
6. ROI math sketched: typical 18–36 month payback via DSO + scrap + productivity
7. Baseline establishment: 90 days post-go-live before measuring against pre-rollout (warning against month-1 ROI reporting)
8. Reporting cadence: monthly CFO dashboard from OIE; quarterly steering committee
9. Leading vs lagging indicators distinction: training % leading, OTD improvement lagging
10. "What 'failure' looks like" explicitly named: OTD flat at 6 months, DSO not improving, CAPAs piling up
11. Warning: do NOT measure ROI in months 1–6 (system still being implemented)
12. Acknowledgment that some value (audit-readiness, sale-attractiveness, operational discipline) is hard to quantify financially but real

### Self-check 7 — IATF 16949 audit preparation

**Prompt:** Manufacturer 14 months into Accelerando deployment. IATF 16949 audit scheduled for month 18. Quality Manager asks: "What do we need to do over the next 4 months to be audit-ready?"

**Expected rubric:**
1. Comprehensive QMS module check: every NCR closed with root cause analysis documented
2. Every CAPA has effectiveness check completed and signed off (the "no theater" rule)
3. All open CAPAs have explicit owner + due date; no orphans
4. Internal audit cycle complete: ≥75% of system audited within last 12 months
5. Management review meetings: 4 quarterly meetings minimum, minutes captured in QMS
6. Document control: every procedure has current version, approval signature, distribution log
7. Training records (LMS): 100% of certifications current for active operators; audit trail of training delivery
8. Calibration records: every measurement instrument calibrated within cycle; out-of-cal flagged immediately
9. Supplier quality scorecards active and reviewed
10. Customer complaints: all linked to NCRs, root cause analysis, CAPAs
11. IATF-specific elements: PPAP workflow evidence, control plans linked to BOMs, FMEA documentation, MSA records, APQP for new parts
12. Top NCR categories trending DOWN (data from QMS over months 9–17)
13. PI CoE projects: at least 1 closed with effectiveness sustained 90+ days; demonstrates continuous improvement
14. Customer scorecards from top OEM customers: trending stable or improving
15. Pre-audit simulation: external audit firm runs mock audit in month 16; gap closure by month 17
16. Auditor visit logistics: site tour preparation, document staging, leadership availability

### Self-check 8 — Customer escalation (rescue scenario)

**Prompt:** 220-employee fabricator's top customer (35% of revenue) just issued formal "supplier corrective action" citing quality, delivery, and EDI compliance issues. Customer threatens to re-source within 12 months unless visible improvement. CEO calls: "We're 6 months from go-live on Accelerando but it's not stopping the bleeding. What do we do?"

**Expected rubric:**
1. Pre-month-1 summit: CEO + Quality Manager + COO + relevant customer-facing person to commit publicly to rescue
2. Phase 0 (before formal kickoff): immediate visible actions — daily QMS-style NCR tracking in spreadsheets if needed; weekly progress report to customer
3. Sequence shift: QMS basics deployed FIRST (months 1–4), before ERP core
4. Customer-facing transparency: monthly CAPA progress reports sent proactively to customer's SQE
5. Direct relationship between Quality Manager and customer's SQE (not through sales)
6. Specific quick win identified: label compliance (or whatever EDI/quality issue customer flagged) as first-90-day project with measurable improvement
7. Customer site visit invited at month 6 to see new shop floor
8. Interim Deployment Manager (external) brought in to take operational burden off exhausted COO
9. COO retention plan: stay bonus tied to month 18 audit pass
10. Investment vs. sale question acknowledged: deployment maximizes business value regardless of outcome
11. Cash flow risk acknowledged (chargebacks + investment); 12 months working capital available recommended
12. Talent retention plan during stress: visible progress + acknowledged plan = retention
13. Customer-pressure success metrics tracked: chargeback $, OTD, top customer scorecard rating
14. Realistic timeline communicated to customer: "visible improvement in 90 days, sustained improvement in 12 months, audit-grade discipline in 18"
15. Owner alignment: deployment IS the business strategy, whether selling or holding

### Self-check 9 — Process manufacturer asks for advice

**Prompt:** Family-owned specialty chemicals manufacturer (140 employees, $60M revenue) heard about Accelerando from a peer. They make industrial cleaning compounds — process manufacturing, batch operations, FIFO inventory, lot tracking critical. Asks if Accelerando fits.

**Expected rubric:**
1. Acknowledges Accelerando CAN serve process manufacturing (the ERP module covers batches/lots)
2. Acknowledges this manufacturing skill doc is for DISCRETE; process needs adapted playbook
3. Names the ontology differences:
   - Batches not work orders
   - Recipes not BOMs (with phantom percentage yields)
   - Lots not serial numbers (deeper traceability)
   - Expiration dates everywhere
   - FIFO mandatory
4. Names HACCP (if food adjacency) or specific chemicals-industry standards
5. Recommends: yes engage, but with a consultant/playbook adapted for process manufacturing
6. Acknowledges out of v1.0 scope for the current skill doc; process-manufacturing skill doc is future work
7. Suggests interim approach: engage with the Agicore team / community for process-specific deployment guidance
8. Cautions against: assuming the discrete playbook directly applies; running the deployment with discrete-trained consultants
9. KPIs that DO transfer: OTD, inventory turns, DSO, customer complaint rate, audit findings
10. KPIs that change: first-pass yield → batch yield variance; cycle counting → lot reconciliation; scrap → off-spec disposal

### Self-check 10 — Owner is considering whether to deploy

**Prompt:** Owner of a 165-employee specialty manufacturer ($55M revenue) calls. "I've been on QuickBooks + Excel for 20 years. We're profitable, my customers are happy enough. Why should I spend $700k+ on a deployment that's going to disrupt everything for 18 months?"

**Expected rubric:**
1. Direct engagement with the question (not deflection)
2. Acknowledges legitimate skepticism: many ERP deployments fail; status quo is genuinely viable for some
3. Lists conditions under which DEPLOYMENT IS RIGHT:
   - Growth ambition that current systems can't support
   - Customer demanding capabilities (EDI, traceability) you can't provide
   - Quality or delivery problems eating margin
   - Compliance requirement (IATF, AS9100) that paper systems can't sustain
   - Succession/sale planning (clean operations + auditable systems = transferable value)
   - Personal exhaustion of running operations with workarounds
4. Lists conditions under which DEPLOYMENT IS WRONG:
   - Profitable, stable, no growth ambition, owner planning to operate forever
   - No customer pressure for systems
   - No compliance pressure beyond current state
   - Owner not willing/able to lead change
   - Insufficient working capital for the project + business
   - No identified successor / sale plan that needs auditable operations
5. Quantifies upside: typical 18-30 month ROI via DSO + inventory turns + scrap reduction
6. Quantifies downside: 30%+ of mid-size ERP deployments fail or significantly under-deliver
7. Names risk factors that predict deployment success vs failure (sponsorship, change-mgmt, master data)
8. Acknowledges: "if you're not sure, don't" — half-committed deployments fail
9. Recommends concrete next step: do a 2-week Config intake; if the COO emerges as sponsor-quality, proceed; if not, table for 12 months
10. Honest about the alternative: continuing with QuickBooks + Excel is genuinely viable for some manufacturers; the right answer might be "no, not now"

---

*Accelerando Manufacturing Super Skill Doc v1.0.0 · MIT*
*The deployment is the work. The system is the artifact of the work.*
