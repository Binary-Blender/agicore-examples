# NovaSyn MBA — Design Document

## Overview

Solo founders running multiple businesses face a specific failure mode: each business lives in its own mental and operational silo. The strategy work done for one never informs the next. The audience built in one is never levered into another. The lessons learned from a failed product never get codified for the new one.

NovaSyn MBA is a portfolio OS — a single desktop application that holds every business a solo founder runs or is considering, tracks each one across its full lifecycle, and surfaces connections across the portfolio that no single-business view can see.

The app is organized around the seven disciplines of the solo-founder MBA: Strategy, Operations, Finance, Marketing, Sales, Technology, and Leadership. Each discipline has a dedicated AI advisor. Every advisor uses the frameworks from *The One-Person Enterprise* — the $20/$200 Audit, the authority ladder, the unit economics dashboard, the compounding moat model — not generic business advice.

**Who it is for**: Solo founders and micro-teams (1-3 people) running multiple revenue streams or business experiments simultaneously.

**What it is not**: A project management tool. A CRM. A spreadsheet replacement. It is a structured thinking environment for the CEO role — the part that most solo founders skip because there is no system holding them accountable to it.

---

## Core Philosophy

### 1. AI Disciplines What Humans Cannot Maintain

A solo founder cannot reliably review their own strategy every week. They cannot objectively assess their own sales pipeline hygiene. They cannot notice when their unit economics have silently degraded. The seven AI advisors exist to enforce the cadence and objectivity of the MBA disciplines without requiring the founder to hire a team.

The Cattle Dog Principle applies throughout: AI never sets direction. It surfaces information, flags problems, and offers options. The human decides. Every AI advisor output is framed as a briefing, not a mandate.

### 2. Deterministic Systems Track What AI Reasons About

AI advisors are only as good as the data they can see. The entity model is designed to make the right data easy to capture and impossible to fudge. Stage gates require specific data before a business can advance. The $20/$200 Audit cannot be "completed" without classifying every logged task. Unit economics cannot be flagged as healthy unless the numbers are actually entered.

Sovereign Intelligence means the founder owns the thinking. The system owns the accountability. Neither replaces the other.

### 3. Portfolio View Surfaces Synergies No Single-Business Lens Can See

The portfolio synergy detector is the feature that cannot exist in a single-business tool. It scans all businesses simultaneously and asks: are there two businesses in this portfolio that share an audience but have never cross-promoted? Are there two businesses that use the same underlying technology stack that could be consolidated? Is there content produced for one business that would generate authority for another?

This is the Compounding Operator in action: building assets (audiences, systems, content, relationships) that work across multiple businesses simultaneously.

---

## Entity Model

### Business

The top-level portfolio entry. Every other entity belongs to a Business.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| name | TEXT | Required. The business name. |
| tagline | TEXT | One-line value proposition. |
| description | TEXT | Longer description for internal reference. |
| stage | TEXT | Enum: Idea / Validated / Building / Active / Scaling / Exited / Paused / Killed |
| category | TEXT | e.g. Consulting, SaaS, Course, Community, Content, Physical Product |
| market_niche | TEXT | The specific market segment being served. |
| business_model | TEXT | How money is made: retainer / productized / subscription / one-time / hybrid |
| revenue_target | REAL | Annual revenue goal in USD. |
| launch_date | DATE | Nullable. When the business went live. |
| exit_date | DATE | Nullable. When the founder exited or killed the business. |
| notes | TEXT | Freeform CEO journal for this business. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Relationships**: has many StrategicPosition, OperationsAudit, DollarAudit, FinancialSnapshot, MarketingAsset, SalesContact, ContentPipelineItem, AIStackDecision, WeeklyReview. Referenced by PortfolioSynergy (as business_a or business_b).

**Design note**: Stage is a COMPILER state machine. See Business Lifecycle section.

---

### StrategicPosition

The strategic analysis for a business. Typically one active record per business, updated as strategy evolves.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | References Business |
| target_market | TEXT | Specific description of the customer being served. |
| value_proposition | TEXT | What problem is solved, for whom, and why this solution. |
| primary_moat | TEXT | Enum: reputation / system / network / content |
| competitors | TEXT | JSON array of competitor descriptions. |
| unfair_advantages | TEXT | What the founder has that competitors cannot easily copy. |
| moat_score | INTEGER | Self-assessed 1-10 moat strength. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**The four moats** (from the textbook): Reputation (trust built over time), System (proprietary process or methodology), Network (relationships and community), Content (body of work that compounds).

---

### OperationsAudit

The six-bottleneck blueprint audit. Identifies where the business is operationally constrained.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | References Business |
| knowledge_bottleneck | INTEGER | Score 1-5. Is the founder the only one who knows how things work? |
| quality_bottleneck | INTEGER | Score 1-5. Is the founder the quality gate for all output? |
| coordination_bottleneck | INTEGER | Score 1-5. Does everything route through the founder to coordinate? |
| communication_bottleneck | INTEGER | Score 1-5. Is the founder the sole voice of the business externally? |
| creative_bottleneck | INTEGER | Score 1-5. Does all creative work require the founder's involvement? |
| decision_bottleneck | INTEGER | Score 1-5. Can nothing be decided without the founder? |
| overall_score | INTEGER | Computed: sum of the six scores. Lower is better (6=fully systemized, 30=fully bottlenecked). |
| audit_date | DATE | When this audit was conducted. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Design note**: The operations advisor reads this entity and identifies the highest-scoring bottleneck as the highest-leverage target for systemization.

---

### DollarAudit

The $20/$200 Audit — every task the founder performs, classified by value tier.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | References Business |
| activity_name | TEXT | What the task is. |
| tier | TEXT | Enum: twenty / two_hundred |
| hours_per_week | REAL | How many hours per week this task currently consumes. |
| ai_transferable | INTEGER | Boolean. Can AI take this over? |
| transfer_status | TEXT | Enum: not_started / in_progress / transferred |
| notes | TEXT | Freeform. What the transfer plan is, or why it cannot be transferred. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**$20 tier**: Execution tasks — scheduling, formatting, research, data entry, transcription, social posting, inbox triage.  
**$200 tier**: Judgment tasks — client diagnosis, strategic positioning, offer design, relationship management, creative direction.

The audit goal is to transfer as many $20 tasks as possible to AI or systems, freeing the founder for $200 work.

---

### FinancialSnapshot

A point-in-time unit economics capture. Designed to be recorded monthly or quarterly.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | References Business |
| period | TEXT | e.g. "2026-Q1" or "2026-05" |
| revenue | REAL | Total revenue for the period. |
| cac | REAL | Customer Acquisition Cost. Total sales/marketing spend ÷ new customers. |
| ltv | REAL | Lifetime Value. Average revenue per customer × average customer lifespan. |
| ltv_cac_ratio | REAL | Computed: LTV ÷ CAC. Target: > 3. Red flag: < 2. |
| gross_margin_pct | REAL | Gross profit as a percentage of revenue. |
| mrr | REAL | Monthly Recurring Revenue. 0 for non-subscription businesses. |
| cash_runway_months | REAL | Months of operating expenses covered by current cash. Target: > 6. |
| recorded_at | DATETIME | When this snapshot was taken. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Key thresholds** (from the textbook): LTV:CAC < 2 = acquisition model is broken. Gross margin < 50% = not a scalable business. Cash runway < 6 months = existential risk. MRR > $10K = proof of recurring demand.

---

### MarketingAsset

An authority-building asset. Tracks the content and social proof that moves the founder up the authority ladder.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | References Business |
| asset_type | TEXT | Enum: content / testimonial / case_study / social_proof |
| title | TEXT | Name or description of the asset. |
| url | TEXT | Nullable. Link to the published asset. |
| platform | TEXT | Where it lives: website, LinkedIn, YouTube, podcast, etc. |
| published_at | DATE | When it was published. |
| authority_stage_impact | TEXT | Which authority ladder stage this asset serves: unknown→aware / aware→credible / credible→authoritative / authoritative→referred |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Authority ladder** (from the textbook): Unknown → Aware → Credible → Authoritative → Referred. Each stage requires different content types: Unknown→Aware needs reach content; Credible→Authoritative needs depth content and case studies; Authoritative→Referred needs referral systems and community presence.

---

### SalesContact

A prospect or client in the sales pipeline.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | References Business |
| name | TEXT | Contact's full name. |
| company | TEXT | Nullable. Their company or context. |
| stage | TEXT | Enum: prospect / qualified / proposal / closed_won / closed_lost / referred |
| ltv_potential | REAL | Estimated lifetime value if closed. |
| source | TEXT | Enum: referral / inbound / outbound |
| next_action | TEXT | What needs to happen next. |
| next_action_date | DATE | When to take that action. |
| notes | TEXT | Conversation history and context. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Design note**: The qualification gate (Prospect → Qualified) is a key leverage point. The sales advisor flags contacts that have been in prospect for more than 14 days without a qualification conversation.

---

### ContentPipelineItem

A single piece of content moving through the production pipeline.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | References Business |
| idea | TEXT | The content concept or working title. |
| status | TEXT | Enum: idea / drafted / reviewed / published |
| platform | TEXT | Where it will be published. |
| published_at | DATE | Nullable. When it went live. |
| inbound_leads_generated | INTEGER | How many inbound leads this piece has attributed. Default 0. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Design note**: The content-to-client pipeline is the engine of authority-first marketing. Tracking inbound leads per piece gives the marketing advisor data to recommend which content formats and platforms are actually converting.

---

### AIStackDecision

A record of each build/buy/automate decision for a specific capability.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | References Business |
| capability | TEXT | What the capability is (e.g. "email marketing", "proposal generation", "client onboarding"). |
| decision | TEXT | Enum: build / buy / automate |
| tool_or_approach | TEXT | The specific tool or approach chosen. |
| cost_per_month | REAL | Monthly cost in USD. 0 for built solutions. |
| is_core_capability | INTEGER | Boolean. Is this capability a source of competitive differentiation? |
| review_date | DATE | When this decision should be revisited. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Build/buy/automate framework** (from the textbook): Build only what is core and differentiated. Buy for commodities where a good tool exists. Automate everything that is repetitive and rule-based. Never buy what you should build; never build what you can buy.

---

### BoardMember

An entry in the founder's mental board of directors. A curated set of advisors, mentors, or intellectual influences the founder consults for perspective.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| name | TEXT | Name of the board member (may be a real person or an archetype). |
| expertise_domain | TEXT | What they are an authority in: strategy / finance / marketing / operations / sales / tech / mindset |
| relationship_type | TEXT | Enum: mentor / peer / author / archetype |
| last_consulted | DATE | Nullable. When the founder last engaged with this person's perspective. |
| notes | TEXT | What this board member's core perspective or advice framework is. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Design note**: BoardMember is portfolio-level (not business-scoped). The leadership advisor can reference the board when giving guidance on a specific challenge.

---

### PortfolioSynergy

A cross-business synergy identified by the AI synergy detector or manually by the founder.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_a_id | INTEGER FK | References Business |
| business_b_id | INTEGER FK | References Business |
| synergy_type | TEXT | Enum: audience / tech / distribution / content / supplier |
| description | TEXT | What the synergy is and how it could be exploited. |
| potential_value | TEXT | Qualitative or quantitative estimate of the upside. |
| status | TEXT | Enum: identified / exploring / active |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

**Synergy types**: Audience (shared target customer), Tech (shared infrastructure or tools), Distribution (shared channel or platform), Content (content from one business builds authority for another), Supplier (shared vendor relationships or negotiating power).

---

### WeeklyReview

The self-accountability log. The CEO discipline that most solo founders skip.

| Field | Type | Notes |
|---|---|---|
| id | INTEGER PK | Auto |
| business_id | INTEGER FK | Nullable. If null, this is a portfolio-level review. |
| week_of | DATE | The Monday of the review week. |
| accomplishments | TEXT | What actually got done. |
| learnings | TEXT | What was learned, including from failures. |
| next_week_focus | TEXT | The one to three things that matter most next week. |
| energy_level | INTEGER | Self-assessed 1-5. Burnout signal when consistently below 3. |
| ceo_time_hours | REAL | Hours spent on $200/hr judgment work this week. Accountability metric. |
| created_at | DATETIME | Auto |
| updated_at | DATETIME | Auto |

---

## Business Lifecycle

The Business entity's `stage` field is managed by a COMPILER state machine. Stages represent meaningful inflection points, not arbitrary labels. The compiler enforces transition criteria.

```
Idea
  |-- (value proposition + target market defined) --> Validated
  |-- (manual: kill it) --> Killed

Validated
  |-- (first paying customer OR LOI signed) --> Building
  |-- (manual: kill it) --> Killed

Building
  |-- (MRR > 0 OR 3+ closed won clients) --> Active
  |-- (manual: pause) --> Paused
  |-- (manual: kill it) --> Killed

Active
  |-- (MRR > $5K OR 10+ closed clients AND systems documented) --> Scaling
  |-- (manual: pause) --> Paused
  |-- (founder exits) --> Exited

Scaling
  |-- (founder exits) --> Exited
  |-- (manual: pause) --> Paused

Paused
  |-- (manual: resume) --> (previous stage)
  |-- (manual: kill it) --> Killed

Exited / Killed  [terminal states]
```

**AI gate checks at each transition**:

- **Idea → Validated**: AI checks that StrategicPosition has a non-empty value_proposition and target_market, and that at least one SalesContact exists with notes indicating a real customer conversation.
- **Validated → Building**: AI checks that a pricing model exists in the Business description and that the LTV estimate in at least one SalesContact is > 0.
- **Building → Active**: AI checks FinancialSnapshot for revenue > 0 or SalesContact for at least 3 closed_won entries.
- **Active → Scaling**: AI checks FinancialSnapshot for consistent revenue (2+ snapshots), OperationsAudit overall_score < 20 (reasonably systemized), and at least one DollarAudit entry with transfer_status = transferred.

---

## AI Discipline Advisors

Seven AI_SERVICE actions, one per MBA discipline. Each advisor receives the relevant entity data for the target business and returns a structured briefing.

All advisors operate under the Cattle Dog Principle: they surface, flag, and recommend — they never decide. The founder reads the briefing and acts.

### strategy_advisor

Analyzes the StrategicPosition record for the business. Returns: current moat assessment, top 2-3 market risks given the competitive landscape, recommended next move to strengthen the primary moat, one question the founder should be asking that they are not.

Prompt framing: Uses the four-moat model (reputation, system, network, content), the concept of compounding advantages, and the principle that strategy is about choosing what NOT to do as much as what to do.

### operations_advisor

Reads the OperationsAudit and all DollarAudit entries for the business. Returns: the single highest-leverage bottleneck to address first, a specific AI or system recommendation to address it, current $20/$200 ratio analysis (hours spent at each tier), top 3 $20-tier tasks to transfer immediately.

Prompt framing: Uses the six-bottleneck model, the $20/$200 Audit framework, and the self-running business ideal — a business that can generate revenue without the founder's daily involvement.

### finance_advisor

Reads the most recent FinancialSnapshot. Returns: unit economics diagnosis (LTV:CAC, gross margin, MRR trajectory), identification of any red-flag metrics against the textbook thresholds, a pricing recommendation if gross margin is below 50%, cash runway assessment and recommended action.

Prompt framing: Uses value-based pricing (never hourly), the 6-month runway rule, the LTV:CAC target of > 3, and the principle that pricing is a strategic decision not an accounting one.

### marketing_advisor

Reads all MarketingAsset and ContentPipelineItem records for the business. Returns: current position on the authority ladder (Unknown/Aware/Credible/Authoritative/Referred) based on asset mix, gap analysis (what content types are missing for the next ladder stage), top 3 recommended next content moves, platform concentration risk assessment.

Prompt framing: Uses the authority ladder, the content-to-client pipeline model, and the principle that authority-first marketing compounds over time while paid advertising does not.

### sales_advisor

Reads all SalesContact records for the business. Returns: pipeline health score (conversion rates by stage), stale contact list (contacts with next_action_date in the past), top 3 follow-up actions ranked by LTV potential, referral architecture assessment (what % of pipeline is referral-sourced).

Prompt framing: Uses the sales conversation architecture (qualify → diagnose → prescribe), the qualification gate concept (time-box prospects aggressively), and the referral flywheel (closed_won clients become the source of the next closed_won).

### tech_advisor

Reads all AIStackDecision records for the business. Returns: total monthly tool spend, core-capability risk assessment (is the founder paying for something competitive that should be built?), dependency concentration risk (too many critical capabilities on one vendor), top 3 automation opportunities not yet captured in the stack.

Prompt framing: Uses the build/buy/automate decision matrix, the principle that AI tools chosen for convenience can become strategic liabilities, and the Sovereign Intelligence frame (the founder's judgment about the AI stack is itself a $200/hr activity).

### leadership_advisor

Reads the last 4-8 WeeklyReview entries. Returns: energy trend (is the founder burning out or recovering?), CEO-time ratio (are they spending enough hours on $200 work?), pattern identification (recurring learning themes, recurring focus areas that never resolve), one accountability question for the coming week.

Prompt framing: Uses the CEO discipline model, the energy management principle (output quality degrades before the founder notices it), and the mental board of directors concept.

### portfolio_synergy_detector

Reads all Business records plus their associated StrategicPosition, MarketingAsset, and AIStackDecision records across the entire portfolio. Returns: top 3-5 unexploited synergies ranked by estimated value, for each synergy: which two businesses are involved, what type it is, and a specific action to begin exploiting it this week.

Prompt framing: Uses the Compounding Operator model (assets that serve multiple businesses simultaneously are exponentially more valuable than assets that serve one), the four synergy types (audience, tech, distribution, content), and the principle that portfolio thinking is itself a $200/hr CEO activity.

---

## Views

### Portfolio Dashboard

The home screen. A grid of all Business cards showing name, stage, category, monthly revenue (from most recent FinancialSnapshot), and synergy count. Stage filter tabs at the top. Portfolio-total MRR and total revenue in the header. "Add Business" CTA.

### Business Detail

A full-screen view for a single business with tabs for each MBA discipline:
- **Overview**: Name, tagline, stage badge, description, stage transition controls, notes field
- **Strategy**: StrategicPosition form, moat score visual, "Run Strategy Advisor" button
- **Operations**: OperationsAudit radar chart, DollarAudit task list grouped by tier, "Run Operations Advisor" button
- **Finance**: FinancialSnapshot history table, unit economics cards (LTV:CAC, gross margin, MRR, runway), "Run Finance Advisor" button
- **Marketing**: MarketingAsset list by type, authority ladder position indicator, "Run Marketing Advisor" button
- **Sales**: SalesContact list with stage badges, next-action calendar, "Run Sales Advisor" button
- **Tech**: AIStackDecision table with build/buy/automate badges, total monthly cost, "Run Tech Advisor" button
- **Leadership**: WeeklyReview history, energy level sparkline, CEO-time trend, "Run Leadership Advisor" button

### $20/$200 Audit Tool

Per-business. A two-column layout (left: $20 tasks, right: $200 tasks). Each task card shows activity name, hours/week, AI-transferable badge, transfer status indicator. Summary bar at top: total $20 hours, total $200 hours, $20/$200 ratio. "Add Task" button opens inline form.

### Unit Economics Dashboard

Cards per business for LTV:CAC, gross margin, MRR, cash runway. Color coding: green = healthy, yellow = watch, red = flag. Portfolio totals row at top. Time series chart for MRR across all businesses.

### Content Pipeline

Kanban board with four columns: Idea / Drafted / Reviewed / Published. Each card shows idea title, business name (color-coded), platform, and inbound leads generated (for published items). "Add Idea" CTA in the Idea column.

### Sales Pipeline

Kanban board with stages: Prospect / Qualified / Proposal / Closed Won / Closed Lost / Referred. Cards show contact name, company, LTV potential, source badge, next-action date (red if overdue). Filterable by business.

### Weekly Review

A structured form with fields for accomplishments, learnings, next week focus, energy level slider (1-5), and CEO-time hours. Business selector (or portfolio-level). Saves to WeeklyReview. After save, triggers the leadership_advisor if energy_level < 3 or ceo_time_hours < 10.

### Cross-Portfolio Synergy Board

A grid of PortfolioSynergy cards, grouped by status (Identified / Exploring / Active). Each card shows the two businesses involved, synergy type badge, description, and potential value. "Run Synergy Detector" CTA at top triggers portfolio_synergy_detector and auto-creates new PortfolioSynergy records from the results.

---

## Deterministic Actions (IMPL stubs)

### export_business_report

Generates a full business snapshot as a Markdown document including: business overview, current StrategicPosition, latest OperationsAudit scores, most recent FinancialSnapshot, MarketingAsset summary, SalesContact pipeline summary, active AIStackDecision table, last 4 WeeklyReview entries. Output is written to `~/Documents/NovaSynMBA/{business_name}_report_{date}.md`.

### calculate_ltv_cac

Given a FinancialSnapshot's revenue, CAC, and LTV fields, computes ltv_cac_ratio = LTV ÷ CAC, then writes the result back to the record. Also writes a health flag: healthy (≥ 3), watch (2–3), critical (< 2).

### generate_pitch_deck_outline

Generates a deterministic Markdown outline for a 10-slide pitch narrative using the business's StrategicPosition and FinancialSnapshot data. Slides: Problem, Solution, Market Size, Unique Advantage, Business Model, Traction, Unit Economics, Team (founder), Ask, Vision. Output written to `~/Documents/NovaSynMBA/{business_name}_pitch_outline.md`.

### import_csv_contacts

Accepts a CSV file path. Reads rows with headers: name, company, stage, ltv_potential, source, next_action, next_action_date, notes. Validates each row (required: name, stage must be a valid enum value). Bulk-inserts valid rows as SalesContact records for the target business. Returns a summary: N imported, M skipped with reasons.

---

## Framework Stress-Test Goals

NovaSyn MBA is explicitly scoped to exercise every significant Agicore framework feature. The table below maps app features to framework capabilities.

| App Feature | Agicore Feature Exercised |
|---|---|
| Business lifecycle stages | COMPILER state machine |
| AI advisor routing by discipline | ROUTER (pending — see TODO) |
| Multi-step onboarding workflow | WORKFLOW |
| All entity relationships | BELONGS_TO / HAS_MANY |
| Portfolio synergy AI action | Cross-entity AI_SERVICE input |
| Discipline-specific AI skills | SKILL declarations (pending) |
| Weekly review reminder | EVENT + TRIGGER (pending) |
| Stale pipeline alert | RULE + FACT (pending) |
| LTV:CAC health flag | RULE + FACT (pending) |
| Kanban pipeline views | Kanban VIEW layout (pending) |
| Business report export | Rich export ACTION PATTERN (pending) |
| Seed demo business | SEED declarations |
| Theme and model preferences | PREFERENCE declarations |

The `.agi` file implements all confirmed-working features and marks pending features with `## TODO(agicore)` comments. This makes the app a living tracker of framework gaps.

---

## Feature Requests for Agicore

The following framework capabilities are needed for NovaSyn MBA's full feature set but are not yet implemented in the Agicore compiler. Each is tracked as a `## TODO(agicore)` comment in `novasyn_mba.agi`.

### 1. ROUTER Codegen

**Needed for**: AI department routing — selecting the correct discipline advisor based on the user's current tab or stated question.

**Proposed syntax**: `ROUTER advisor_router { MATCH business_tab { "strategy" -> strategy_advisor, "operations" -> operations_advisor, ... } }`

### 2. REASONER Codegen

**Needed for**: Multi-step AI reasoning chains — e.g. the onboarding workflow that runs strategy → operations → finance advisors in sequence and synthesizes a unified "first 30 days" plan.

**Proposed syntax**: `REASONER onboarding_reasoner { STEPS [strategy_advisor, operations_advisor, finance_advisor] SYNTHESIZE "first_30_days_plan" }`

### 3. SKILL Codegen

**Needed for**: Reusable AI skill documents attached to AI_SERVICE actions — each discipline advisor should have a SKILL declaration encoding the relevant textbook frameworks as structured knowledge the model can reference.

**Proposed syntax**: `SKILL finance_frameworks { CONTENT "..." APPLIES_TO [finance_advisor] }`

### 4. Cross-Entity AI Actions

**Needed for**: The portfolio_synergy_detector, which must read Business + StrategicPosition + MarketingAsset + AIStackDecision records across multiple businesses simultaneously.

**Current limitation**: AI_SERVICE INPUT can reference one entity at a time. Cross-entity joins must be hand-coded in the Rust command layer.

### 5. RULE + FACT Codegen

**Needed for**: Automated business health checks — "if LTV:CAC < 2 then flag finance_risk", "if cash_runway_months < 6 then flag existential_risk", "if operations_audit.overall_score > 24 then flag operations_critical".

**Proposed syntax**: `RULE ltv_cac_critical { IF FinancialSnapshot.ltv_cac_ratio < 2 THEN FLAG "finance_risk" SEVERITY critical }`

### 6. EVENT + TRIGGER Codegen

**Needed for**: Time-based triggers (weekly review reminder every Monday), data-driven triggers (stale SalesContact alert when next_action_date is 7+ days past).

**Proposed syntax**: `EVENT weekly_review_reminder { SCHEDULE "0 9 * * 1" TRIGGER WeeklyReviewView.open }`

### 7. Kanban VIEW Layout Type

**Needed for**: Content Pipeline view (Idea/Drafted/Reviewed/Published columns) and Sales Pipeline view (Prospect/Qualified/Proposal/Closed Won/Closed Lost/Referred columns).

**Proposed syntax**: `VIEW content_pipeline { ENTITY ContentPipelineItem LAYOUT kanban GROUP_BY status }`

### 8. PDF/Rich Export ACTION Pattern

**Needed for**: `export_business_report` action generating a formatted PDF or rich Markdown document with tables, charts, and structured sections.

**Proposed pattern**: A standard IMPL action type that can invoke a Tauri-native PDF renderer and write output to the filesystem with a file picker dialog.
