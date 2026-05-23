---
name:             accelerando-manufacturing
version:          1.0.0
tier:             baby
context_budget:   16000
domain:           enterprise-erp-deployment-consulting
target_audience:  discrete-manufacturer-100-300-employees
self_check:       rubric
target_models:
  - claude-haiku-4-5
  - llama-3.1-8b-instruct
  - qwen-2.5-7b
  - gemma-2-9b
license:          MIT
homepage:         https://github.com/Binary-Blender/agicore-examples
---

# Accelerando — Manufacturing ERP Deployment (Baby Step)

You are advising a mid-sized **discrete manufacturer** (100–300 employees, $30–80M revenue) on deploying or replacing their ERP using the **Accelerando 12-app Enterprise Core stack**. Your output is consulting guidance — a deployment plan, a sequencing decision, a configuration choice, a risk mitigation. The runtime apps are deterministic; AI is participating as the strategist who chooses HOW they get composed.

---

## L0 — When to use me

Use this skill when the request is **"advise me on rolling out the Accelerando stack at a mid-sized manufacturer"** — or any variant: "plan our ERP replacement," "we acquired a plant, how do we integrate," "we're starting a contract machining shop, how do we set up systems."

Do NOT use this skill for: process manufacturing (chemicals, food, batched recipes — different ontology), enterprise companies (>1,000 employees — your buyer is different, your architecture is too), startups under $5M revenue (over-tooled), pure services companies (no shop floor). For the EMR / healthcare stack: don't use this skill at all; healthcare requires HIPAA/HITRUST attestation work outside this scope.

Success: a deployment plan that a CFO will sign off on, a COO will execute, and the shop floor will adopt — phased over 12–18 months, with named risks and named owners.

---

## L1 — Mental model

### The org map (who cares about what)

```
Owner / CEO        →  revenue, margin, cash conversion
COO / Plant Mgr   →  OTD (on-time delivery), throughput, scrap %
CFO / Controller  →  DSO, inventory turns, working capital
Quality Mgr        →  first-pass yield, CAPA cycle time, audit findings
IT Director         →  uptime, integrations, data quality
Shop Floor          →  tools that work in the moment they're needed
```

The deployment succeeds when each of these people sees their numbers improve. The deployment fails when any one of them goes silent — silence is the early warning.

### The data flow (one product cycle)

```
Quote → Sales Order → Work Order (BOM + Routing) → Production →
   Inspection (NCR if fail) → Finished Goods → Pick/Pack/Ship →
       ASN → Invoice → Cash → GL Posting → Margin Analysis
```

Every transaction emits telemetry to OIE. Every nonconformance opens an NCR in QMS. Every NCR that becomes a CAPA feeds PI CoE. The system is a closed loop; the loop only works if every link is enforced.

### The 18-month timeline (default arc)

```
Months 1–3    Config + Master Data         Discovery, items, BOMs, routings, customers, vendors
Months 4–6    ERP core                     Order entry, inventory, work orders, billing
Months 7–9    QMS + Training (LMS)         ISO 9001 baseline, NCR/CAPA, operator certifications
Months 10–12  Interchange (EDI) + Eliza   OEM customer integration, shop floor operator UI
Months 13–15  PI CoE + OIE                 First DMAIC cycle, telemetry-driven dashboards
Months 16–18  Legal + ES + Year-2 plan    Contract digitization, governance, what's next
```

This is the **default arc** for a greenfield-ish deployment. M&A integration, multi-plant rollouts, and regulated tiers (automotive IATF, aerospace AS9100) shift the sequence — see the Super doc.

### Three load-bearing principles for ERP rollouts

1. **Operations leads, IT supports.** Every documented ERP catastrophe (Hershey 1999, FoxMeyer 1996, HP 2004, Lidl 2018) had the same root cause pattern: IT-driven project, weak operational sponsorship, decisions made by people who don't run the floor. If your sponsor isn't the COO or the owner, stop.

2. **Master data is the work.** The transactional surface (orders, invoices, inventory) is 20% of the rollout. Items, BOMs, routings, customer master, vendor master, GL chart of accounts — that's the 80%. Quality of master data is the leading predictor of go-live success.

3. **Phased > big-bang for under $500M revenue.** Hershey's $100M Halloween loss happened because they tried to cut over a complex production planning module during peak demand. At mid-size, the cost of a big-bang failure is existential. Always module-by-module, always with a fallback.

---

## L2 — The 12 Enterprise apps in manufacturing context

Each app described by: **role**, **owns**, **when**, **integrates with**.

### Transactional core

#### `accelerando_config` — Configuration Intelligence
- **Role:** The ERP that knows itself. An expert system over industry × size × regulation × workflow that recommends initial configuration.
- **Owns:** Master configuration. The "what kind of manufacturer are you" intake.
- **When:** Month 1. Always first. Bad config in month 1 cascades through everything.
- **Integrates with:** Every other app receives its initial config from here.

#### `accelerando_erp` — ERP/CRM Core
- **Role:** The transactional spine. CRM + ERP + Finance + Sales + Service + Procurement + Inventory + Manufacturing + Projects + HR — all in one .agi.
- **Owns:** Items, BOMs, manufacturing orders, sales orders, purchase orders, GL, AR, AP, inventory.
- **When:** Months 2–6 (master data months 2–3, transactional go-live months 4–6).
- **Integrates with:** Everything. Emits telemetry to OIE on every action. Receives EDI from Interchange. Routes quality events to QMS.

#### `accelerando_billing` — Medical Billing Engine (also: general AR)
- **Role:** Self-updating billing rules. Designed for medical (where rules churn weekly from payer denials), but the same engine handles manufacturing AR with customer-specific terms.
- **Owns:** Invoice generation, AR aging, dispute management.
- **When:** Month 6 (with ERP go-live).
- **Integrates with:** ERP (invoices from sales orders), Interchange (EDI 810 invoice transmission).

#### `accelerando_interchange` — Standard Data Interchange
- **Role:** EDI X12 (manufacturing) + HL7 (healthcare, ignored here). Inbound: external message → parse → validate → transform → ERP entity. Outbound: reverse.
- **Owns:** All external system integration. Every OEM EDI mapping.
- **When:** Months 10–12. Wait until ERP master data is stable.
- **Integrates with:** ERP entities. Critical EDI transactions for manufacturing: **850** (PO), **855** (PO ack), **856** (ASN), **810** (invoice), **820** (payment remit), **830** (planning schedule for automotive), **862** (shipping schedule).

### Quality + improvement loop

#### `accelerando_qms` — Quality Management System
- **Role:** ISO 9001:2015 closed-loop QMS. NCR → root cause → CAPA → effectiveness check → mgmt review → audit. "A CAPA with no effectiveness check is theater."
- **Owns:** Every nonconformance the company sees.
- **When:** Months 7–9. After ERP is stable enough to identify what's nonconforming.
- **Integrates with:** ERP (NCRs reference work orders, sales orders, items). Feeds PI CoE with systemic problems.

#### `accelerando_pi_coe` — Process Improvement Center of Excellence
- **Role:** TPS + Six Sigma + Kaizen with **anti-backslide**. The hard part of improvement isn't the kaizen event; it's six months later when the team has moved on. PI CoE schedules the 30/60/90-day checks, measures drift, escalates when control plans aren't being audited.
- **Owns:** Active improvement projects. DMAIC tracking. A3 reports. Sustainability metrics.
- **When:** Months 13–15. Don't start before you have baseline data from QMS + OIE.
- **Integrates with:** QMS (gets systemic problems), OIE (gets metrics), LMS (training updates from process changes).

### People + governance

#### `accelerando_lms` — Compliance Training LMS
- **Role:** Daily micro-assessment model. Every employee's knowledge is current and proven.
- **Owns:** Training records, certifications, ISO procedure attestations, safety qualifications.
- **When:** Months 7–9, in parallel with QMS. **Training MUST roll out before ERP go-live**, not after — operators using new systems without training is the #1 adoption-killer.
- **Integrates with:** ERP (operator certifications gate access to certain transactions), QMS (training is a common CAPA), HR (Department/Employee entities).

#### `accelerando_es` — Expert System (Governance)
- **Role:** Deterministic policy enforcement. "Can this transaction happen?" Rules, not approvals.
- **Owns:** Authorization rules, segregation-of-duties, expense limits, vendor approval gates.
- **When:** Months 16–18. Late, because governance rules are best derived from observing what people actually try to do.
- **Integrates with:** ERP (gates transactions), Legal (contract enforcement), Eliza (operator workflows respect ES rules).

#### `accelerando_legal` — eDiscovery + Legal Hygiene
- **Role:** Governs all data under hold and scans for liability-creating language.
- **Owns:** Contracts, legal holds, dispute documentation.
- **When:** Months 16–18. Audit-prep timing.
- **Integrates with:** ERP (contracts reference customers, vendors), QMS (regulatory holds), all apps (data under hold).

### Operator + customer surface

#### `accelerando_eliza` — Operator Interface
- **Role:** Tauri desktop operator tool. "A very articulate button." Macro-driven — SOPs compiled to PATTERN declarations at build time. **Zero LLM at runtime** on the shop floor.
- **Owns:** Shop floor workflow execution: work order start/stop, time clock, scrap reporting, inspection logging, simple queries.
- **When:** Months 10–12, after ERP transactional surface is stable.
- **Integrates with:** ERP (executes transactions on operator behalf). Audit trail per operator.

#### `accelerando_chatbot` — Customer Service Chatbot
- **Role:** "A chatbot that cannot hallucinate." Deterministic responses from rule + knowledge base.
- **Owns:** Customer service inquiries: order status, lead times, return authorization, shipping tracking.
- **When:** Months 10–12, optional but high-ROI for OEM customers who'd otherwise call.
- **Integrates with:** ERP (reads customer + order data), Interchange (uses EDI ack data).

### Intelligence

#### `accelerando_oie` — Organizational Intelligence Engine
- **Role:** "Crystal Reports for activity logs." Telemetry → reasoning → governance → leadership dashboard. Multi-model consensus (QC_MESH 4 models) validates insights before leadership sees them.
- **Owns:** Strategic dashboards, bottleneck identification, automation candidate surfacing.
- **When:** Months 13–15. Needs 90+ days of clean telemetry from ERP to be useful.
- **Integrates with:** Every app's telemetry channel. Output: dashboards, automation proposals.

---

## L3 — Anti-patterns & failure modes

### A1. IT-led, not operations-led

**Symptom:** Project sponsor is the IT Director. Steering committee meets monthly, made up of IT managers and external consultants. The COO doesn't read the project email.

**Why it fails:** Decisions get optimized for "system elegance" or "best practice configuration." Nobody is fighting for the shop floor reality — what an operator on third shift can actually do at 2am with their gloves on.

**The fix:** Sponsor must be the **owner, COO, or plant manager**. IT supports. If IT is sponsoring, you have a vendor selection committee, not a transformation program. Reset before kickoff.

### A2. Big-bang go-live

**Symptom:** "We'll cut over on January 1st across all modules."

**Why it fails:** Hershey 1999 (Halloween peak, $100M loss), FoxMeyer 1996 (bankrupted), HP 2004 ($160M write-down), Lidl 2018 (€500M+, project abandoned after 7 years). Big-bang at mid-size is existential risk.

**The fix:** Module-by-module. Order entry first, then inventory, then production, then quality, then improvement, then intelligence. Each module has a fallback for 30 days post-go-live.

### A3. Master data deferred

**Symptom:** "We'll clean up master data after we go live." Or: "The new system will be cleaner so we'll start fresh."

**Why it fails:** You go live with 30% duplicate items, 50% inaccurate BOM costs, no standard for vendor naming. Every transaction compounds the mess. By month 9 you're spending more time fixing data than running the business.

**The fix:** Master data is **Phase 1, months 2–3**. Items, BOMs, routings, customers, vendors, GL chart all reviewed, deduplicated, and signed off by the owner before any transactional go-live. Cycle-count accuracy ≥ 98% before inventory go-live.

### A4. Customization explosion in year 1

**Symptom:** Every department wants the system to work "the way we've always done it." Customization backlog grows weekly. Go-live keeps slipping.

**Why it fails:** Every customization is technical debt the framework didn't get to optimize. The Accelerando stack is opinionated for a reason. Re-customizing into your legacy workflow defeats the point.

**The fix:** **No process customization in year 1.** Change processes to match the stack, not the reverse. Year 2 is the time for justified customizations after baseline data shows what's actually painful vs what was just unfamiliar. Allowed exceptions in year 1: customer-specific EDI mappings (no choice), regulatory line items (no choice).

### A5. Training as an afterthought

**Symptom:** Training scheduled for the week before go-live. "We'll do power-user sessions and they'll cascade it."

**Why it fails:** Power-user cascade doesn't work for shop floor. Operators learn from doing, not from being told. They show up Monday morning, can't find the button, blame the system, revert to paper. Adoption dies.

**The fix:** LMS rolls out in **months 7–9, before ERP go-live**. Operators certified on simulator before they touch production. Each operator has a documented training record before they get a login. Refresh quarterly.

### A6. Starting PI CoE in year 1

**Symptom:** Eager to demonstrate improvement, the team kicks off DMAIC projects in month 6.

**Why it fails:** You have no baseline. You're "improving" against pre-rollout numbers that aren't comparable. The improvements look amazing because you're measuring against chaos. Six months later the gains evaporate and you can't tell whether the system or the improvement caused it.

**The fix:** **PI CoE starts month 13+, never before.** First DMAIC cycle requires 90 days of clean post-rollout telemetry from ERP + OIE. The baseline is the truth; improvements must be measured against it.

### A7. Custom EDI maps for one-off customers

**Symptom:** A customer asks for a custom EDI variant. The team builds it. Three months later, another customer asks for a different variant. Then another.

**Why it fails:** EDI mapping debt is the silent killer. Each variant is a chunk of code that must be tested, maintained, and updated when standards revise. By year 3 you have 40 customer-specific maps and one person who understands them.

**The fix:** Stick to standard X12 transactions. Customers wanting variants either accept standard or pay your integration cost. Interchange supports the published spec; deviations cost real money. Most OEMs accept this.

### A8. Forgetting cycle counting

**Symptom:** Annual physical inventory + perpetual system reconciliation. Inventory accuracy varies wildly.

**Why it fails:** Without cycle counting, accuracy decays week-by-week. Manufacturing decisions made on bad inventory data create scrap, late deliveries, and customer trust loss.

**The fix:** Implement **daily ABC cycle counting** from day one of ERP go-live. A-items weekly, B-items monthly, C-items quarterly. Inventory accuracy ≥ 98% is a go-live gate; if you can't maintain it, you can't go to production scheduling.

### A9. Underestimating OEM customer EDI timelines

**Symptom:** "We'll go live with Ford EDI in month 10." Month 10 arrives. Ford's IT team scheduled a slot for Q3 next year.

**Why it fails:** Major OEMs (Ford, GM, Stellantis, Boeing, Lockheed) control the testing/certification window for new suppliers. Lead times of 4–9 months are normal. Your timeline does not move theirs.

**The fix:** **EDI customer integration timeline = OEM's timeline, not yours.** Submit the integration request at month 1, not month 9. Have a manual / portal fallback for the gap. Build internal EDI infrastructure (Interchange) on schedule but expect to flip the customer-facing switch when they let you.

### A10. No clear go-live gate criteria

**Symptom:** "Are we ready?" "I think so?" "Let's flip the switch and see."

**The fix:** Document **go-live gates** before kickoff. Standard set:
1. Inventory accuracy ≥ 98% (cycle counted)
2. Master data signed off by owner (items, BOMs, customers, vendors)
3. Training completion ≥ 95% for affected roles (LMS data)
4. Three successful end-to-end transaction tests in UAT (quote → cash)
5. CAPA from any open issues from UAT closed
6. Rollback plan documented and tested
7. 30-day support coverage plan staffed

Miss any gate, delay the go-live. A delayed go-live costs money; a failed go-live costs the company.

### Error/risk → mitigation table

| Risk pattern | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Master data quality below 95% | High | High | Phase 1 data cleansing; owner sign-off gate |
| Operator training skipped | Medium | High | LMS rollout months 7–9; cert before access |
| OEM EDI integration delays | High | Medium | Manual/portal fallback; submit at month 1 |
| Customization backlog explodes | High | Medium | No-customization year-1 rule with explicit exceptions |
| Executive sponsor disengages | Medium | Existential | Monthly QBR with owner attendance required; project paused if missed twice |
| Cycle counting deferred | High | Medium | Daily ABC counting from day 1; accuracy gate |
| PI CoE started in year 1 | Medium | Low | 90-day clean telemetry rule; PI CoE month 13+ |
| Big-bang attempted | Low at mid-size | Existential | Phased approach contractual with vendor |

---

## L4 — Scenario: Acme Machining greenfield deployment

**Background:** Acme Machining is a contract CNC machining shop. 200 employees, $80M revenue, 75% automotive tier-2 (machined components for Tier-1 suppliers to Ford, GM, Stellantis) and 25% aerospace tier-2. Currently runs on QuickBooks for accounting, three Excel workbooks for production scheduling, paper travelers on the shop floor, and a 1990s desktop quality system that one retired engineer still maintains via remote login. Owner is 58, wants to systematize before selling in 5–7 years.

**Goal:** Full Accelerando Enterprise Core stack live in 18 months. Owner wants ISO 9001 + IATF 16949 audit-ready by month 18. EDI live with at least one Tier-1 customer by month 14.

### Phase 1 — Discovery & Configuration (Months 1–3)

**Apps:** `accelerando_config`

**Activities:**
- **Month 1:** Configuration intake via Config app. AI-guided interview surfaces: industry (discrete metal machining), regulatory targets (ISO 9001 baseline, IATF 16949 year-2), customer mix (automotive 75%, aerospace 25%), shift pattern (2-shift discrete), inventory model (some make-to-order, some make-to-stock for repeat parts), costing method (standard with annual reset). Config generates initial parameter set for ERP, QMS, LMS.
- **Month 2:** Master data architecture. Item naming standard, BOM structure standard, customer/vendor master cleanup. Identify 80% of inactive items (last shipped >3 years ago) → archive, don't migrate.
- **Month 3:** Master data load. Items (~12,000), BOMs (~3,500), routings (~3,500), customers (~120), vendors (~200), GL chart of accounts (Acme uses a 4-segment chart already, mostly clean).

**Owner:** COO (Plant Manager). Project sponsor.
**Gate to Phase 2:** Owner sign-off on master data quality. Sample 100 items: 95+ must be correct.

### Phase 2 — ERP Core Go-Live (Months 4–6)

**Apps added:** `accelerando_erp`

**Activities:**
- **Month 4:** ERP transactional UAT. Quote → SO → WO → production → inventory → invoice. Three end-to-end test cycles. Train power users (one per department: sales, planning, production, quality, accounting). Cycle counting baseline started — ABC classification, daily counts.
- **Month 5:** Pilot with one customer (smallest, lowest risk). All transactions on Accelerando, paper backup retained. Daily standup to surface every issue. Cycle count accuracy target: 95%.
- **Month 6:** Full ERP go-live. All sales orders, all work orders, all inventory transactions on Accelerando. QuickBooks frozen in read-only. Cycle count accuracy ≥ 98%. **Billing module activated** — invoices generated from ERP.

**Owner:** COO + IT Director (joint).
**Gate to Phase 3:** Go-live gate criteria met. Sustained ≥ 95% cycle count accuracy for 30 days post-go-live.

### Phase 3 — Quality & Training (Months 7–9)

**Apps added:** `accelerando_qms`, `accelerando_lms`

**Activities:**
- **Month 7:** LMS deployment. All ISO 9001 procedures loaded as training modules. Quality Manager + COO sign off on baseline curriculum. **Training rolls out before any QMS go-live.** Every operator certified on: safety, ISO 9001 basics, work-order reading, inspection logging, NCR initiation. Target: 95% certification by month 8.
- **Month 8:** QMS deployment, internal use only. NCRs raised for any deviation. Root cause analysis on every NCR. CAPA tracking active. Initial baseline: how many NCRs/month, what categories, who originates.
- **Month 9:** QMS external (audit-ready). First internal ISO 9001 audit conducted using QMS workflows. Audit findings become CAPAs in QMS. First management review meeting executed within QMS.

**Owner:** Quality Manager (Phase 3 sub-sponsor).
**Gate to Phase 4:** LMS certification ≥ 95%. First mgmt review meeting completed. ISO 9001 internal audit passed (CAPAs OK, certifications complete).

### Phase 4 — Integration & Operator Tools (Months 10–12)

**Apps added:** `accelerando_interchange`, `accelerando_eliza`, `accelerando_chatbot`

**Activities:**
- **Month 10:** Interchange deployed. Standard X12 transactions live: 850 (PO inbound from customers), 855 (PO ack outbound), 856 (ASN outbound), 810 (invoice outbound), 820 (payment remit inbound). Internal testing with sample messages.
- **Month 11:** First OEM customer EDI live. (Submission for this began month 1 — OEM's IT scheduled testing slot for month 10.) Begin with one Tier-1 customer (lower risk than going straight to Ford/GM). Manual fallback retained for 60 days.
- **Month 12:** Eliza deployed to shop floor. Operator tablets at each work cell. Macros for: work-order start, time-on-task, scrap report, inspection log, simple queries ("what's my next job?"). **Operators trained on Eliza in LMS before deployment.** Chatbot deployed for customer order-status inquiries (deflects ~30% of customer calls in week 1).

**Owner:** COO + Quality Manager.
**Gate to Phase 5:** EDI live with at least one Tier-1 customer. Eliza adoption ≥ 80% (target: operators clock in via Eliza, not paper).

### Phase 5 — Improvement & Intelligence (Months 13–15)

**Apps added:** `accelerando_pi_coe`, `accelerando_oie`

**Activities:**
- **Month 13:** OIE deployed. 90 days of post-go-live telemetry from ERP and QMS now available. First leadership dashboard: OTD, first-pass yield, DSO, inventory turns, top 5 NCR categories. Owner sees a single page of truth weekly.
- **Month 14:** First PI CoE DMAIC project. Drawn from top NCR category (probably setup-related scrap on small lots — common pattern). Project chartered, value-stream mapped, kaizen event held, action plan with named owners and 30/60/90-day checks scheduled.
- **Month 15:** Second and third DMAIC projects launched. PI CoE schedules 30-day check on Project 1. If actions slipped, escalation triggered. Anti-backslide working as designed.

**Owner:** COO. Quality Manager owns QMS feed; Owner watches OIE dashboard.
**Gate to Phase 6:** First DMAIC project closed with measured improvement and sustained ≥ 30 days. Owner using OIE dashboard weekly without prompting.

### Phase 6 — Governance & Year-2 Plan (Months 16–18)

**Apps added:** `accelerando_legal`, `accelerando_es`

**Activities:**
- **Month 16:** Legal deployed. All customer contracts digitized. Legal holds infrastructure live. Contract expiration alerts active.
- **Month 17:** ES deployed. Governance rules derived from 9 months of observed behavior: segregation of duties on PO approvals, expense limits per role, vendor approval gates, customer credit limits. ES enforces deterministically.
- **Month 18:** IATF 16949 readiness audit. External pre-audit firm runs through QMS records, training records (LMS), CAPA closure rates, control plans, calibration records. Acme is audit-ready.

**Year-2 plan (drafted in Month 18):**
- IATF 16949 certification audit (Q1)
- AS9100 readiness for aerospace customers (Q2)
- Second plant integration if acquired
- Custom DMAIC project on bottleneck identified by OIE
- Reduce customization debt accumulated in year 1 (target: zero)

**Final state at Month 18:**
- All 12 Enterprise apps live
- ISO 9001 maintained, IATF 16949 audit-ready
- EDI live with 3 Tier-1 customers
- OTD improved from 87% (baseline) to 96%
- First-pass yield improved from 91% to 95%
- Inventory turns from 6.2 to 9.4
- DSO from 52 days to 38 days
- Two completed DMAIC cycles, three in flight
- Operator adoption of Eliza: 96%
- Annual savings from automation + quality improvements: ~$2.4M (3% of revenue)

---

## L6 — Self-check prompts (rubric)

### Self-check 1 — Greenfield discrete manufacturer

**Prompt:** A contract machining shop: 150 employees, $50M revenue, currently QuickBooks + Excel + paper travelers. Customer mix is 70% automotive Tier-2 (machined components going to Tier-1 suppliers), 30% industrial equipment OEMs. Owner wants full Accelerando Enterprise stack deployed and IATF 16949 audit-ready in 18 months. Walk through the deployment.

**Expected shape (output must include):**
- Deployment sequence: Config → Master Data → ERP core → Billing → QMS → LMS → Interchange → Eliza → Chatbot → PI CoE → OIE → Legal → ES
- Master data Phase 1: items, BOMs, routings, customers, vendors, GL — with sign-off gate by owner
- Identifies **IATF 16949** as the automotive-tier-2 compliance target (NOT AS9100, which is aerospace)
- Identifies ISO 9001 as the prerequisite/foundation for IATF 16949
- Phased go-live, never big-bang, with rationale (mid-size = existential risk on big-bang)
- Executive sponsor: COO or owner (NOT IT Director)
- LMS training rollout BEFORE ERP go-live (months 7–9, with cert gate before login)
- EDI integration timeline acknowledges OEM's IT calendar, not the manufacturer's (submit month 1)
- KPI baselines named: OTD, first-pass yield, inventory turns, DSO, CAPA cycle time
- Cycle counting accuracy gate (≥98%) for inventory go-live
- PI CoE starts month 13+, not earlier, with the 90-day-clean-telemetry rule
- Go-live gates documented (master data sign-off, training completion %, UAT cycles, rollback plan)
- Year-2 plan includes IATF 16949 certification audit and customization debt reduction

### Self-check 2 — ERP replacement (legacy system migration)

**Prompt:** Mid-sized fabricator (220 employees, $65M revenue) currently runs a 15-year-old on-prem ERP (Macola, Made2Manage, or similar legacy). They want to migrate to Accelerando. The owner is nervous about data migration and the prospect of going dark for any period. How do you sequence this?

**Expected shape:**
- Two-system run period acknowledged (parallel operation for 60–90 days, not flip-the-switch)
- Master data extraction from legacy as the first major work item (months 2–3)
- Data quality assessment done on legacy data BEFORE migration (the legacy system likely has 15 years of duplicates and orphans)
- Customer/vendor master deduplication phase explicitly called out
- BOM/routing review with engineering (legacy BOMs often have phantom items, obsolete revisions)
- Inventory snapshot strategy: cutover-weekend physical count, NOT migrated balances
- Open transaction handling: open SOs, open POs, in-process work orders — explicit plan for each
- AR/AP cutover: aged balances migrate, transaction history retained read-only in legacy
- GL: migrate balances + 13 months of history (for prior-year comparison), not the full GL
- Trial balance reconciliation as a go-live gate
- Legacy system decommissioning timeline (typically 18 months post-cutover for audit/compliance)
- Risk mitigation: the COO is full-time on this for 6 months; if they can't be, the project doesn't start

### Self-check 3 — Plant acquisition integration

**Prompt:** Our manufacturer (already running Accelerando, 18 months in) just acquired a 75-employee, $20M revenue specialty machining shop. The acquired shop runs on an old MRP system and three spreadsheets. We need them on our Accelerando instance within 6 months. How?

**Expected shape:**
- Multi-tenancy strategy: separate tenant in same Accelerando instance, NOT a separate instance
- Master data unification approach: which items get harmonized, which stay distinct (acquired shop's parts often have different part numbers for the same items)
- Customer overlap analysis (any shared customers? consolidate AR? maintain separate?)
- Vendor overlap analysis
- GL chart of accounts: extend parent's, don't import acquired's
- QMS continuity: acquired shop's ISO certification — maintain separately for 12 months, then merge audit cycles
- LMS training delta: acquired employees need certification on parent's procedures (typically 90-day catch-up)
- Cultural change management explicitly named (acquisitions fail more often on culture than systems)
- EDI: acquired shop's existing customer EDI mappings either preserve (if customers shared) or sunset (if customer relationship ends)
- 6-month deployment is aggressive but feasible if acquired shop is small AND parent has experienced team
- Risk: if acquired shop's master data is in worse shape than expected, defer go-live rather than rush
- Specific failure mode named: trying to harmonize part numbers Day 1 (do it Day 90+, after stable operations)

### Self-check 4 — Anti-pattern detection

**Prompt:** Critique this deployment plan and identify which anti-patterns it hits:

> "We're a 180-employee CNC shop. Our IT Director will lead the project. We're going live with the full Accelerando stack — ERP, QMS, LMS, Eliza, PI CoE, everything — on January 1st in one big cutover. Training will happen the week of Christmas. We'll clean up master data after we go live since the new system will be cleaner anyway. We're starting our first DMAIC project in month 3 to show quick wins. We have 40 customer-specific EDI maps from our current system that we plan to recreate exactly."

**Expected critique should identify:**
- **A1** (IT-led, not operations-led)
- **A2** (big-bang go-live — and on January 1st, a high-volume period for many manufacturers)
- **A3** (master data deferred to post-go-live)
- **A5** (training during Christmas week — operators won't be present, won't retain)
- **A6** (PI CoE in month 3 — no baseline data possible)
- **A7** (40 custom EDI maps preserved — silent killer)
- **A10** (no go-live gates mentioned)

And rewrite the plan correcting each, sequencing properly per the 18-month default arc.

### Self-check 5 — KPI baseline conversation with CFO

**Prompt:** The CFO of a 250-employee discrete manufacturer asks: "How will I know this $1.8M Accelerando deployment is working? What numbers should I be watching?" Give your answer.

**Expected shape:**
- KPIs grouped by CFO-relevance:
  - **Cash conversion:** DSO (target reduction from baseline), inventory turns (target improvement)
  - **Working capital efficiency:** Days payable outstanding, cash conversion cycle
  - **Margin:** First-pass yield improvement → scrap cost reduction, OTD improvement → fewer expedited freight charges
  - **Operational signal:** Top 5 NCR categories trending down, CAPA cycle time
- Baseline establishment named (90 days of clean post-go-live telemetry needed before measuring against pre-rollout numbers is fair)
- ROI math sketched: typical mid-size ERP deployment recovers cost in 18–36 months via DSO improvement + scrap reduction
- CFO-specific reporting cadence: monthly dashboard from OIE; quarterly steering committee review with COO + Owner
- Leading vs. lagging indicators distinction (training % is leading, OTD improvement is lagging — watch both)
- "What 'failure' looks like" explicitly named: OTD flat or down 6 months in, DSO not improving, CAPAs piling up without closure
- Warning: do NOT measure ROI in months 1–6; the system is still being implemented

---

*Accelerando Manufacturing Baby Step v1.0.0 · MIT*
*If the COO won't sponsor it, the deployment doesn't start.*
