# Accelerando Clinical

**The chart that catches what you're about to miss.**

> A major drug-drug interaction that clears with a click is not safety.  
> Safety is: the allergy conflict blocks the order.  
> Safety is: the controlled substance cannot proceed without a current PDMP query.  
> Safety is: the critical result notifies the provider — and tracks whether they acknowledged it.

---

## What Most Clinical Systems Miss

The EHR is not the problem. The problem is that the EHR will warn you and let you proceed anyway.

This system has three categories of clinical safety:

**Hard blocks** — the workflow cannot proceed:
- Allergy conflict on new medication order
- Controlled substance ordered without current PDMP query
- Critical result released to patient before provider acknowledgement
- Auto-release of a critical lab value to patient portal

**Escalations** — the provider sees it, must document a response:
- Major drug-drug interaction
- Renal dose adjustment required
- STAT order unfulfilled at 60 minutes

**Reminders** — visible at encounter start, cleared by action:
- Unsigned note at 24 hours
- Pending result unacknowledged at 72 hours
- Screenings due (mammogram, colorectal, lung, depression, diabetes)
- Open care gaps from population health

---

## The Allergy Block

```
RULE critical_allergy_alert {
  WHEN SafetyContext.known_allergy_conflict == true
  THEN FLAG "allergy_conflict_medication_blocked_provider_override_required"
  SEVERITY critical
  PRIORITY 100
}
```

`CheckAllergyConflict` runs using RxNorm cross-sensitivity rules — not just exact drug name matches. A patient with penicillin allergy and a documented cephalosporin cross-sensitivity triggers the block when amoxicillin-clavulanate is ordered, even if the order uses a brand name. The provider can override with a documented reason. The override is in the chart. If the patient has a reaction, the chart shows the override decision.

---

## The Controlled Substance Block

```
RULE controlled_without_pdmp {
  WHEN SafetyContext.controlled_no_pdmp_check == true
  THEN FLAG "controlled_substance_ordered_without_pdmp_query_blocked"
  SEVERITY critical
  PRIORITY 100
}
```

PRIORITY 100. The clinical system will not allow a controlled substance medication order to proceed — regardless of DEA schedule — without a current PDMP query on file. "Current" defaults to 3 days. The pharmacy system enforces the same rule at point of dispense. Two checkpoints, same requirement.

---

## Critical Results — The Loop That Must Close

```
RULE critical_result_unreviewed {
  WHEN ChartContext.unreviewed_critical_results == true
  THEN FLAG "critical_result_not_acknowledged_immediate_provider_review"
  SEVERITY critical
  PRIORITY 100
}
```

When a lab or radiology result is marked critical:
1. CriticalResultPacket fires to clinical and patient portal channels
2. The result is held — it cannot auto-release to the patient portal
3. The system tracks acknowledgement — who reviewed it, when
4. If not acknowledged within 1 hour: escalation flag

Critical results never auto-release. The patient portal `ReleaseResult` action checks the review status and blocks if the provider hasn't cleared it.

---

## AI at Build Time — Provider Assistance, Not Replacement

Three AI actions assist providers at encounter time:

**`GenerateClinicalNote`** — drafts a SOAP note from encounter data: chief complaint, vitals, problem list, medication list, and orders placed. The provider reviews, edits, and signs. The AI draft is a starting point — the provider signature is the attestation.

**`SuggestDifferential`** — given chief complaint, vitals, and history, returns a ranked differential with ICD-10 codes and reasoning. The provider chooses. The system records the chosen diagnosis.

**`GenerateReferralLetter`** — generates a specialist referral letter with clinical summary, reason for referral, and relevant results. Eliminates the blank-page problem for referrals.

All three are deterministic at runtime — the AI output is captured in the chart, the provider's decision is recorded, and the signed document is immutable.

---

## Screening Engine — At the Point of Care

The screening checks run at encounter open, not in a weekly report:

```
RULE lung_cancer_eligible_unscreened {
  WHEN ScreeningContext.lung_cancer_screening_eligible_unscreened == true
  THEN FLAG "lung_cancer_screening_eligible_patient_not_screened_ldct_indicated"
  SEVERITY warning
  PRIORITY 70
}
```

The USPSTF criteria run at encounter open. The provider sees screening alerts when the patient is in the room — not two weeks later in a report. The provider can order, document a declination, or schedule for a future visit. The alert clears when one of those actions is taken.

---

## Architecture

```
accelerando_clinical.agi
│
├── ENTITY × 12
│   Patient, Encounter, Problem, Medication, Allergy
│   Vital, ClinicalNote, Order, Result, Referral, CarePlan, Immunization
│   ClinicalAlert
│
├── STAGES
│   Encounter → scheduled → checked_in → provider_review → in_progress
│             → pending_orders → pending_sign → signed → closed
│   Order → ordered → sent → pending → resulted → acknowledged → cancelled
│   ClinicalNote → draft → pending_cosign → signed → addended
│
├── PACKET × 5
│   NewEncounterPacket → billing, population health
│   PrescriptionPacket → pharmacy (exactly_once)
│   ReferralPacket → scheduling
│   CriticalResultPacket → radiology, patient portal (exactly_once)
│   CCDPacket → interchange, patient portal
│
├── MODULE × 5
│   ChartEngine      → critical results, unsigned notes, med reconciliation
│   SafetyEngine     → allergy, drug-drug, PDMP, renal/hepatic, duplicate therapy
│   CPOEEngine       → order fulfillment tracking, STAT escalation
│   VitalsEngine     → hypertensive emergency (PRIORITY 100), critical vitals
│   ScreeningEngine  → USPSTF: mammogram, colorectal, lung, depression, diabetes, immunizations
│
├── REASONER × 1
│   clinical_intelligence_reasoner → weekly: safety alert patterns, screening gaps, documentation quality
│
├── WORKFLOW × 6
│   open_encounter, record_vitals, order_medication
│   sign_note, generate_ccd, acknowledge_result
│
└── ACTION × 22
    Deterministic: CreateEncounterRecord, CheckAllergyConflict, CheckDrugInteractions,
    CheckPDMPRequirement, CheckRenalHepaticDose, RecordVitals, EvaluateVitalSigns,
    CheckScreeningDue, ValidateNoteCompleteness, SignClinicalNote, AcknowledgeResult,
    ReleaseResultToPortal, CompileChartData
    AI (claude-sonnet): GenerateClinicalNote, SuggestDifferential,
    GenerateReferralLetter, GenerateCCDDocument
```

---

## Connected to Everything

Clinical is the hub:

- → **Pharmacy**: `PrescriptionPacket` on every medication order
- → **Radiology**: receives `FinalReportPacket` for result acknowledgement
- → **Scheduling**: `ReferralPacket` for specialty routing
- → **Billing**: `NewEncounterPacket` for charge capture
- → **Population Health**: encounter data for care gap closure, risk scoring
- → **Patient Portal**: results, CCD, critical notifications
- ← **Population Health**: care gap alerts presented at encounter open
- ← **Pharmacy**: `PDMPHighRiskPacket` when pharmacy flags a patient
