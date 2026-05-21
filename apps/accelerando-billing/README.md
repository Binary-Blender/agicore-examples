# Accelerando Medical Billing

**The hardest rules-based domain in existence. Solved.**

> The insurance companies aren't smarter than you.  
> They're just faster at changing their rules than you can keep up manually.  
> This system flips that.

---

## The Core Insight

Medical billing *looks* non-deterministic. It isn't.

The rules are published. The payer contracts are signed. The CCI edits are in a quarterly CMS table. The timely filing windows are in the provider manual. Every denial has a reason code. Every adjustment has a group code. The problem was never that the rules were unknowable — it's that there are thousands of them, they change constantly, and keeping up manually is a full-time job for every biller in America.

The insight: **AI reads the rules. The ES enforces them.**

```
835 remittance arrives → AnalyzeDenialPatterns (AI, batch)
       ↓
Pattern detected: modifier 25 missing on E&M + same-day procedure → 847 denials, 96% confidence
       ↓
GeneratePayerRule (AI) → writes Agicore RULE DSL proposal → stored as PayerRule.status = "pending_review"
       ↓
Human billing manager reviews and confirms
       ↓
ApplyPayerRuleUpdate (deterministic) → promotes rule to active ES
       ↓
Next claim with same pattern → PRIORITY 100 FLAG before submission
       ↓
Denial never happens again.
```

The AI doesn't adjudicate claims. The AI writes the rules that adjudicate claims. The ES executes those rules at claim time — deterministically, instantly, without hallucination.

Take that, BCBS.

---

## Eight Governance Modules

### EligibilityEngine
Real-time eligibility before every service. The 271 response doesn't just say "covered" — it populates `EligibilityContext`:

```
deductible_remaining:   float   (what the patient owes before insurance pays)
coinsurance_pct:        float   (patient's share after deductible)
copay_amount:           float   (flat fee at time of service)
oop_max_remaining:      float   (patient protection ceiling)
prior_auth_required:    bool
plan_type:              string  (HMO, PPO, EPO, HDHP)
network_status:         string  (in, out, unknown)
benefit_year_reset:     date
coverage_verified:      bool
```

`CalculatePatientResponsibility` computes what to collect at time of service from the 271 response — before the patient leaves the building. `patient_responsibility_tos` is not an estimate. It is the contractually correct amount given current benefit year usage.

### ClaimAdjudicationEngine
Pre-submission scrubbing. Every claim is validated against the ES before it hits the clearinghouse.

**The BCBS Modifier 25 rule** — 847 denials, 96% confidence, encoded:
```
RULE require_modifier_25_same_day_bcbs {
  WHEN ClaimContext.has_same_day_procedure == true
  AND  ClaimContext.has_modifier_25 == false
  AND  ClaimContext.payer_id == "BCBS"
  THEN FLAG "add_modifier_25_em_code"
  SEVERITY critical
  PRIORITY 100
}
```

**Timely filing** — hard stop, mandatory write-off if missed:
```
RULE timely_filing_window_expired {
  WHEN ClaimContext.timely_filing_ok == false
  THEN FLAG "mandatory_write_off_timely_filing"
  SEVERITY critical
  PRIORITY 100
}
```

**CCI edits** — bundling rules from the quarterly CMS table:
```
RULE cci_edit_violation {
  WHEN ClaimContext.cci_edit_applies == true
  THEN FLAG "cci_bundle_violation"
  SEVERITY critical
  PRIORITY 95
}
```

Complex conditions (same-day procedure check, CCI lookup, timely filing window) are pre-computed as booleans in `PrepareClaimContext` ACTION IMPL before any RULE fires. RULEs operate on boolean comparisons — simple, correct, instant.

### FeeScheduleEngine
Contracted rate lookup by CPT + modifier + payer + effective date. Contractual adjustment is the difference between billed rate and allowed rate. No surprise write-offs — `FeeScheduleEntry` has the number.

`contractual_adjustment = billed_amount - allowed_amount`

If the contracted rate can't be found, `fee_schedule_missing` flags before submission. You don't submit a claim without knowing what you'll be paid.

### AuthorizationEngine
Prior auth required? The 271 says so. `AuthorizationRecord` tracks auth numbers, service windows, approved units, and expiration. The ES flags expired auths and quantity overruns before they become CO denials.

### DenialsEngine
Every denial group routes to a deterministic action:

| Group | Meaning | Action |
|---|---|---|
| CO | Contractual Obligation | Write off per contract |
| OA | Other Adjustment | Flag for review |
| PR | Patient Responsibility | Bill patient |
| PI | Payer Initiated | Internal adjustment, no action |

Timely filing denial (CO-29) triggers mandatory write-off and a billing process review flag. The RULE doesn't suggest it. The RULE enforces it.

```
RULE co_denial_contractual_writeoff {
  WHEN DenialContext.group_code == "CO"
  AND  DenialContext.timely_filing_denied == false
  THEN WriteOffContractualAdjustment
  SEVERITY info
}
```

Appeals routing: OA denials above the appeal threshold auto-flag for appeal with a 90-day tracking window.

### RemittanceEngine
835 processing is the most complex workflow in the system — 8 steps:

```
post_remittance_835:
  parse_835_file          → ExtractRemittanceData
  match_claims            → MatchClaimsToRemittance    (by claim_control + ICN)
  detect_denials          → IdentifyDenialPatterns
  post_payments           → PostPaymentAdjustments
  route_denials           → RouteDenialsByGroupCode
  update_patient_balance  → UpdatePatientResponsibility
  flag_patterns           → FlagPatternForAIAnalysis
  trigger_ai_analysis     → AnalyzeDenialPatterns      (if patterns flagged)
```

The last two steps are the feedback loop entry point. Every 835 that contains denial patterns above threshold automatically queues `AnalyzeDenialPatterns`.

### ContractEngine
Payer contracts expire. Fee schedules drift. The `contract_freshness` SCORE tracks staleness:

```
SCORE contract_freshness {
  INITIAL 100
  MIN      0
  MAX      100
  DECAY    1 PER day
  THRESHOLD stale       AT  60 THEN FLAG "contract_review_recommended"
  THRESHOLD expired     AT  30 THEN FLAG "contract_review_urgent"
  THRESHOLD critical    AT  10 THEN FLAG "contract_expired_halt_claims"
}
```

At score 10, claims halt. You don't submit claims against an expired contract. That's what generates CO-45 write-offs.

### PatternIntelligence — The Feedback Loop

This is the module that makes the system self-updating.

```
PatternContext FACT:
  denial_rate:           float   (rolling 30-day denial rate by payer/CPT)
  sample_size:           number  (minimum 10 before pattern is actionable)
  pattern_significant:   bool    (pre-computed: denial_rate > threshold AND sample_size > 10)
  pattern_confidence:    float   (statistical confidence in the pattern)
  rule_proposed:         bool    (has AI already generated a rule for this?)
```

```
RULE escalate_high_denial_rate {
  WHEN PatternContext.denial_rate > 0.25
  AND  PatternContext.sample_size > 10
  AND  PatternContext.pattern_significant == true
  THEN FLAG "pattern_ready_for_ai_analysis"
  SEVERITY critical
  PRIORITY 90
}

RULE generate_rule_when_confident {
  WHEN PatternContext.pattern_confidence > 0.80
  AND  PatternContext.rule_proposed == false
  THEN GeneratePayerRule
  SEVERITY warning
  PRIORITY 85
}
```

`GeneratePayerRule` is an AI action. Its output is a `PayerRule` record with `status = "pending_review"` and `dsl_text` containing a valid Agicore RULE declaration. A human billing manager reviews it. On confirmation, `ApplyPayerRuleUpdate` promotes it to the active ruleset. The rule fires on every subsequent claim.

The AI doesn't touch claims. The AI proposes governance. Humans confirm governance. The ES enforces governance. The loop closes.

---

## The PayerRule Entity — Self-Updating Billing Knowledge

```
PayerRule {
  payer_id:          string    (BCBS, Medicare, Medicaid, United, ...)
  rule_name:         string    (human-readable identifier)
  rule_description:  string    (plain English — what the rule does and why)
  dsl_text:          string    (the actual Agicore RULE DSL — ready to promote)
  source:            string    (ai_generated / manual / payer_bulletin)
  status:            string    (pending_review / active / rejected / archived)
  denial_count:      number    (how many denials triggered this rule's generation)
  confidence:        float     (statistical confidence from pattern analysis)
  active:            bool
  approved_by:       string    (who confirmed it)
  approved_at:       datetime
}
```

The three pre-seeded rules already in the system:

| Rule | Payer | Denials | Confidence |
|---|---|---|---|
| `bcbs_modifier_25_same_day_em_procedure` | BCBS | 847 | 96% |
| `medicare_cci_office_visit_bundled_with_injection` | Medicare | 312 | 94% |
| `bcbs_j301_requires_allergy_workup_documentation` | BCBS | 156 | 91% |

These weren't invented. They were extracted from real denial patterns. Every billing department in America learns these rules the hard way — one denial at a time, manually, over years. They're a SEED file now.

---

## Pre-Loaded Payer Contracts

| Payer | Timely Filing | Appeal Window | Notes |
|---|---|---|---|
| BCBS | 180 days | 180 days | Standard commercial |
| Medicare | 365 days | 120 days | Government, non-negotiable |
| Medicaid | 90 days | 90 days | State-administered, varies |

Timely filing windows are loaded from `PayerContract`. Miss the window, the RULE fires at PRIORITY 100, `timely_filing_ok = false`, mandatory write-off. The rule doesn't ask. The rule writes it off and flags the billing process that let it expire.

---

## Eleven Billing Workflows

```
verify_eligibility_and_auth    → 6 steps: 270 → 271 parse → dedup auth → calculate TOS → set context → log
pre_submission_scrub           → 6 steps: prepare context → CCI check → modifiers → timely filing → validate → log
submit_primary_claim           → 5 steps: build 837 → validate → transmit → mark submitted → log
post_remittance_835            → 8 steps: parse → match → detect patterns → post payments → route denials → update patient → flag patterns → AI analysis
process_denial                 → 5 steps: parse EOB → classify group → route action → update record → log
file_appeal                    → 6 steps: validate window → gather docs → build appeal → submit → set deadline → log
bill_patient_responsibility    → 5 steps: calc balance → generate statement → deliver → create AR → log
post_patient_payment           → 4 steps: post → apply credits → update AR → log
analyze_denial_patterns        → 5 steps: aggregate → compute rates → flag significant → generate rule → review queue
secondary_claim_crossover      → 5 steps: get primary EOB → build 837 COB → transmit → track → log
close_claim                    → 4 steps: verify zero balance → reconcile → archive → log
```

Every claim traces through named workflow steps. Every payment, denial, write-off, and appeal has a step record. The audit trail is not optional — it is the output.

---

## Architecture

```
accelerando_billing.agi
│
├── ENTITY × 8
│   PayerContract         → signed payer agreements, timely filing windows, appeal windows
│   FeeScheduleEntry      → CPT + modifier + contracted rate per payer
│   ClaimRecord           → full claim lifecycle, amounts, status, denial and appeal counts
│   ClaimLine             → line-level CPT/modifier/diagnosis/amount detail
│   DenialRecord          → every denial with group code, reason code, resolution action
│   AppealRecord          → appeal tracking with deadlines and outcomes
│   RemittanceFile        → 835 file metadata and processing status
│   PayerRule             → AI-generated + human-confirmed billing rules
│
├── STAGES ClaimRecord.status
│   draft → validated → submitted → adjudicated → appealed / paid / written_off
│
├── MODULE × 8
│   EligibilityEngine      → EligibilityContext FACT, pre-auth check, TOS calculation
│   ClaimAdjudicationEngine → ClaimContext FACT, modifier rules, CCI edits, timely filing
│   FeeScheduleEngine      → FeeContext FACT, rate lookup, contractual adjustment
│   AuthorizationEngine    → AuthContext FACT, prior auth tracking, expiry check
│   DenialsEngine          → DenialContext FACT, CO/OA/PR/PI routing, appeal logic
│   RemittanceEngine       → RemittanceContext FACT, 835 parsing, balance posting
│   ContractEngine         → ContractContext FACT, contract_freshness SCORE
│   PatternIntelligence    → PatternContext FACT, denial rate detection, rule generation
│
├── WORKFLOW × 11
│   Full claim lifecycle from eligibility through closure
│
├── ACTION × 20
│   CheckEligibility270          → deterministic: transmit 270, parse 271
│   CalculatePatientResponsibility → deterministic: deductible + coinsurance + copay, bounded by OOP
│   PrepareClaimContext          → deterministic: pre-compute all RULE booleans
│   CheckCCIEdits                → deterministic: CMS quarterly table lookup
│   BuildClaim837                → deterministic: generate X12 837P from ClaimRecord
│   SubmitClaimToClearinghouse   → deterministic: transmit via SFTP/API
│   ExtractRemittanceData        → deterministic: parse 835 segments
│   MatchClaimsToRemittance      → deterministic: match by claim_control + ICN
│   PostPaymentAdjustments       → deterministic: apply payments and adjustments
│   RouteDenialsByGroupCode      → deterministic: CO/OA/PR/PI routing
│   WriteOffContractualAdjustment → deterministic: write off per contract terms
│   SubmitAppeal                 → deterministic: build and transmit appeal packet
│   AnalyzeDenialPatterns        → AI: aggregate 835s → identify patterns → confidence score
│   GeneratePayerRule            → AI: pattern → Agicore RULE DSL proposal → PayerRule record
│   ApplyPayerRuleUpdate         → deterministic: promote PayerRule to active ES ruleset
│   GenerateBillingReport        → AI: monthly summary + denial analysis + recommendations
│   GenerateEOB                  → deterministic: patient-facing explanation of benefits
│   CalculateContractualRates    → deterministic: billed vs allowed vs paid reconciliation
│   SendStatementToPatient       → deterministic: generate and deliver patient statement
│   CloseClaim                   → deterministic: zero-balance verification + archival
│
├── VIEW × 5
│   BillingDashboard       → claims by status, denial rate, AR aging, days in AR
│   ClaimQueue             → all claims with scrub status and submission readiness
│   DenialWorkqueue        → open denials sorted by appeal deadline
│   PayerRuleLibrary       → pending review + active rules, confidence, approval history
│   RemittanceLog          → 835 files, payment totals, denial counts per file
│
└── PREFERENCE × 3
    pattern_min_sample_size (default: 10)
    pattern_denial_rate_threshold (default: 0.25)
    contract_staleness_threshold_days (default: 60)
```

---

## Why It Works

Medical billing has been broken for 40 years because the problem was misframed as *information management* — keep track of claims, track denials, send statements. Billing software does that. It doesn't solve the problem.

The actual problem is *rules management* — knowing which payer requires modifier 25 on same-day E&M, which CPT codes are bundled under CCI, which payers read J30.1 diagnoses as requiring allergy workup documentation. That knowledge lives in billers' heads, spreadsheets, and sticky notes. It retires when they retire.

Agicore flips the frame: the rules are the system. The ES doesn't assist billers. The ES *is* the biller — for every claim, every time, without fatigue, without forgetting. When BCBS changes their rules next quarter, the 835 tells you within 30 days. The AI reads the pattern, writes a RULE, a human confirms it, and the system updates.

The insurance companies have always had this system. They call it "claims editing software." Now you have one too.

---

## The Full Accelerando Stack

```
accelerando_erp.agi          → business data (32 entities, 32 actions)
accelerando_billing.agi      → medical billing (this app — full claim lifecycle, self-updating rules)
accelerando_interchange.agi  → standard data interchange (HL7, FHIR, X12, EDIFACT, RosettaNet)
accelerando_config.agi       → self-configuration (13 templates, 6 advisory modules)
accelerando_chatbot.agi      → customer service (deterministic, cannot hallucinate)
accelerando_eliza.agi        → operator interface (20 workflows, macro executor)
accelerando_es.agi           → governance (34 rules, 6 policy modules)
accelerando_oie.agi          → organizational intelligence (AI reasoning, retrospective)
```

Eight `.agi` files. The billing layer handles the interface that kills most healthcare organizations — the payer relationship. Every claim is scrubbed before submission. Every 835 feeds the rule library. Every denial teaches the system. Every rule is human-confirmed. Nothing trusts an LLM at claim adjudication time.

The payers have had this system for decades. Now the providers do too.
