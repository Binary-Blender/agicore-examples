# Accelerando Population Health

**Finding the patient who hasn't called in three years. Before they call 911.**

> The patient with uncontrolled diabetes who stopped coming in isn't non-compliant.  
> They're lost. Lost to the practice. Still on the panel. Still your patient.  
> Population health is finding them before the ER does.

---

## What Population Health Actually Does

Most practices have reports. A report that shows 47 patients are overdue for colorectal screening isn't population health — it's a list. Population health is what happens after the list: who gets called first, what happens when they don't respond, and whether the intervention actually closed the gap.

This system manages the full loop:

```
Care gap identified (HEDIS logic)
       ↓
Priority scored (clinical urgency × patient risk tier × measure impact)
       ↓
Outreach campaign launched (personalized AI messaging per patient)
       ↓
Response tracked — appointment scheduled or not
       ↓
Appointment closes the gap
       ↓
HEDIS measure updates in real time
```

No lists. No manual follow-up. The system tracks outreach attempts, records responses, and escalates to provider review when three attempts produce no response.

---

## Risk Stratification — Finding the Rising Risk

```
RULE high_risk_no_care_manager {
  WHEN RiskContext.high_risk_no_care_manager == true
  THEN FLAG "high_risk_patient_not_enrolled_in_care_management"
  SEVERITY critical
  PRIORITY 93
}

RULE post_discharge_no_followup {
  WHEN RiskContext.post_discharge_no_followup == true
  THEN FLAG "post_discharge_followup_not_scheduled_30_day_readmission_risk"
  SEVERITY critical
  PRIORITY 95
}
```

`StratifyRiskPanel` computes four scores for every patient on the panel:
- **RAF Score** — CMS risk adjustment factor from HCC coding
- **Charlson Comorbidity Index** — validated mortality predictor
- **ED Utilization Risk** — predicted probability of ED visit in 12 months (0–100)
- **Avoidable Admit Risk** — predicted probability of avoidable hospitalization (0–100)

Patients crossing the high-risk threshold without a care manager trigger a PRIORITY 93 flag. Post-discharge patients without a 7-day follow-up appointment fire at PRIORITY 95 — the 30-day readmission window is the most expensive miss in population health.

---

## Disease Registries — The Six That Matter

```
RULE diabetes_uncontrolled {
  WHEN RegistryContext.diabetes_uncontrolled == true
  THEN FLAG "diabetes_registry_patient_a1c_uncontrolled_intensify_management"
  SEVERITY warning
  PRIORITY 85
}
```

Six pre-seeded disease registries:
- **Diabetes** (E10, E11, E13) — tracks A1C, control status, A1C overdue flag
- **Hypertension** (I10–I13) — tracks blood pressure, uncontrolled flag
- **CHF** (I50) — tracks BMP/BNP recency
- **COPD** (J43, J44) — tracks pulmonary follow-up
- **CKD** (N18.3–N18.6) — tracks eGFR trend, declining trajectory flag
- **High Risk Panel** — risk tier = "high" across all conditions

Each registry entry is updated from encounter data as it flows through. A patient with declining eGFR is flagged for nephrology referral consideration before they reach stage 4. A CHF patient overdue for BMP and BNP fires before the next fluid overload event.

---

## HEDIS Quality Measures — Real Time, Not Retrospective

```
SCORE care_gap_closure_rate {
  INITIAL   0 / MIN 0 / MAX 100 / DECAY 0 PER day
  THRESHOLD excellent  AT 85 THEN FLAG "care_gap_closure_excellent_performance"
  THRESHOLD below      AT 60 THEN FLAG "care_gap_closure_below_target_intensify_outreach"
  THRESHOLD critical   AT 50 THEN FLAG "care_gap_closure_critical_quality_measure_at_risk"
}
```

6 pre-seeded quality measures:
- **HEDIS CDC-HbA1c Control <8%** — target 70%, 75th percentile benchmark 72%
- **HEDIS Colorectal Cancer Screening** — target 68%
- **HEDIS Breast Cancer Screening** — target 72%
- **HEDIS Controlling High Blood Pressure** — target 65%
- **HEDIS FUH — Follow-Up After Hospitalization** — target 58%
- **MIPS Annual Influenza Vaccine** — target 80%

Measures recalculate whenever encounter data arrives — not at month end. When a care gap closes (patient completes screening, A1C comes back controlled), the measure updates immediately. Leadership sees live performance, not 30-day-old data.

---

## HCC Recapture — The RAF Revenue at Risk

```
RULE hcc_recapture {
  WHEN QualityContext.hcc_recapture_due == true
  THEN FLAG "hcc_codes_require_annual_recapture_risk_adjustment_revenue_at_risk"
  SEVERITY warning
  PRIORITY 83
}
```

HCC codes must be documented every plan year to count for risk adjustment. `IdentifyHCCRecaptureDue` finds diagnoses documented in prior years that haven't been recaptured yet. `EmitHCCRecapturePackets` sends them to the clinical system — the provider sees the recapture alerts at the next encounter for that patient.

If a CHF patient with a prior-year HCC code doesn't have the code documented this plan year, that RAF weight doesn't count. For a practice with 500 CHF patients, unrecaptured HCC codes are a revenue variance that's entirely preventable.

---

## Architecture

```
accelerando_population_health.agi
│
├── ENTITY × 8
│   PopulationCohort, CareGap, DiseaseRegistry
│   QualityMeasure, RiskScore, OutreachCampaign
│   CareManagementEnrollment, HCCCode
│
├── STAGES
│   CareGap → open → outreach_sent → appointment_scheduled → closed / expired
│   OutreachCampaign → draft → active → paused → completed
│   CareManagementEnrollment → enrolled → active → on_hold → discharged
│
├── PACKET × 4
│   CareGapOpenedPacket → clinical, patient portal
│   HighRiskPatientPacket → clinical, governance/ES
│   QualityMeasureReportPacket → governance/ES
│   HCCRecapturePacket → clinical
│
├── MODULE × 4
│   CareGapEngine        → closure rate score, no-outreach flag, critical gap escalation
│   RiskEngine           → high-risk enrollment rule, post-discharge 30-day rule, care management contact
│   QualityMeasureEngine → below-target flag, 50th percentile escalation, reporting deadline
│   DiseaseRegistryEngine → registry-specific control rules (diabetes, HTN, CHF, CKD, COPD)
│
├── REASONER × 1
│   population_intelligence_reasoner → weekly: care gap trends, risk distribution, measure performance, outreach ROI
│
├── WORKFLOW × 6
│   identify_care_gaps, stratify_risk, compute_quality_measures
│   outreach_campaign, enroll_care_management, hcc_recapture
│
└── ACTION × 18
    Deterministic: RunHEDISCareGapLogic, ScoreCareGapPriority, StratifyRiskPanel,
    AssignRiskTiers, EnrollHighRiskPatients, CalculateQualityMeasures, IdentifyMeasureGaps,
    IdentifyOutreachTargets, ExecuteOutreachCampaign, AssessCareManagementEligibility,
    CreateCareManagementEnrollment, AssignCareManager, IdentifyHCCRecaptureDue
    AI (claude-haiku): GenerateOutreachMessages — personalized per patient and channel
    AI (claude-sonnet): GeneratePopulationReport — executive health summary
```

---

## Seed Data

6 disease registry cohorts + 6 quality measures pre-seeded. The organization adds patients, the registries populate from encounter data, the measures calculate from the registry. Day one coverage with no manual setup.

---

## Connected to the Stack

- ← **Clinical**: encounter data updates registries, closes care gaps, captures HCC codes
- ← **Scheduling**: no-show data for risk scoring (repeated no-shows increase ED risk score)
- → **Clinical**: `HCCRecapturePacket` at next encounter, care gap alerts at encounter open
- → **Patient Portal**: `CareGapOpenedPacket` for patient-facing reminders
- → **Governance/ES**: `HighRiskPatientPacket`, `QualityMeasureReportPacket`
