# Accelerando Radiology

**Critical findings communicated in 60 minutes. Not eventually.**

> A radiologist who finds a pulmonary embolism and files the report is not done.  
> The loop closes when the ordering provider acknowledges the finding.  
> Most RIS implementations track the report.  
> This one tracks the communication.

---

## The Critical Finding Rule

ACR practice guidelines: critical findings must be communicated to the ordering provider within 60 minutes of detection.

```
RULE critical_finding_communication {
  WHEN ReportingContext.critical_finding_not_communicated_60m == true
  THEN FLAG "critical_finding_communication_overdue_60_minutes_ACR_guideline_violated"
  SEVERITY critical
  PRIORITY 100
}
```

When a radiologist signs a report:
1. `IdentifyCriticalFindings` (AI) scans the findings and impression for critical finding language
2. If detected: `CreateCriticalFindingRecord` sets the 60-minute deadline
3. `CriticalFindingAlertPacket` emits immediately to clinical and patient portal
4. `MonitorCriticalFindingAcknowledgement` tracks the clock
5. At 60 minutes without acknowledgement: PRIORITY 100 flag

Critical findings include: pneumothorax, pulmonary embolism, intracranial hemorrhage, aortic dissection, acute fracture in unexpected location, new malignancy on screening exam. The AI does not make the clinical determination — it flags for the radiologist to confirm and communicate. The communication record is in the chart.

---

## Contrast Safety — Two Blocks

```
RULE contrast_allergy_blocked {
  WHEN RISContext.contrast_allergy_no_consent == true
  THEN FLAG "contrast_ordered_known_contrast_allergy_consent_and_premedication_required"
  SEVERITY critical
  PRIORITY 100
}

RULE creatinine_not_checked {
  WHEN RISContext.creatinine_not_checked_contrast == true
  THEN FLAG "iv_contrast_ordered_creatinine_not_checked_AKI_risk"
  SEVERITY critical
  PRIORITY 98
}
```

Both blocks at the scheduling step, before the patient is on the table. Known contrast allergy: premedication protocol required and documented consent obtained. Unknown renal function: creatinine must be checked. The default threshold is 1.5 mg/dL — configurable. Above threshold: nephrology clearance before contrast administration.

---

## DICOM Modality Worklist — Closing the Paper Gap

When imaging is scheduled, `EmitModalityWorklist` sends a `ModalityWorklistPacket` to the DICOM gateway. The modality pulls the patient demographics, accession number, and procedure directly from the worklist. Technologists don't type patient names into the scanner. Accession numbers don't get transposed. The study comes back linked to the correct patient and order.

`accession_number_collision` at PRIORITY 100 — if two orders somehow generate the same accession number, it's caught before images are acquired.

---

## Peer Review — Independence Enforced

```
RULE reviewer_independence {
  WHEN PeerReviewContext.reviewer_independence_violated == true
  THEN FLAG "peer_review_self_review_blocked_same_radiologist_cannot_review_own_study"
  SEVERITY critical
  PRIORITY 100
}
```

`AssignPeerReviewers` enforces independence — a radiologist cannot be assigned to peer review their own study. `SelectStudiesForPeerReview` randomly selects 5% of signed studies per radiologist per month (ACR RADPEER protocol). Major discrepancies escalate to section chief. Clinically significant discrepancies notify the ordering provider.

The `peer_review_concordance` score tracks each radiologist's concordance rate:

```
SCORE peer_review_concordance {
  INITIAL 100 / MIN 0 / MAX 100 / DECAY 0 PER day
  THRESHOLD excellent  AT 95 THEN FLAG "peer_review_concordance_excellent"
  THRESHOLD concerning AT 75 THEN FLAG "peer_review_concordance_concerning_quality_review"
  THRESHOLD critical   AT 60 THEN FLAG "peer_review_concordance_critical_formal_review_required"
}
```

---

## Dose Tracking — The Expensive Miss

```
RULE dose_exceeds_drl_significantly {
  WHEN DoseContext.exceeds_drl_3x == true
  THEN FLAG "dose_exceeds_DRL_3x_immediate_physics_review_required"
  SEVERITY critical
  PRIORITY 97
}
```

Every CT and fluoroscopy study records dose metrics from the DICOM Radiation Dose Structured Report: CTDIvol, DLP, fluoroscopy time, DAP. Each study is compared to the ACR Diagnostic Reference Level for that procedure. Three times the DRL: immediate medical physics review. Any DRL exceedance: technique review.

Cumulative patient dose is tracked across all studies. When a patient's lifetime effective dose is high, the alert prompts clinical discussion about imaging strategy — before ordering the next CT.

---

## AI at Build Time — Radiologist Assistance

**`GenerateRadiologyReport`** — drafts a structured report from study metadata and comparison reports. The radiologist reviews and signs. The draft handles boilerplate (technique, comparison language) so the radiologist focuses on findings.

**`IdentifyCriticalFindings`** — scans the drafted findings and impression for critical finding language patterns. Returns a confidence score. The radiologist decides — the system flags for review, not for auto-escalation.

**`SelectImagingProtocol`** — selects the appropriate imaging protocol given clinical indication, patient characteristics, and contrast plan. Returns protocol parameters and dose targets.

**`GeneratePeerReviewReport`** — generates structured peer review summary from RADPEER score and discrepancy notes.

---

## Architecture

```
accelerando_radiology.agi
│
├── ENTITY × 7
│   ImagingOrder, Study, RadiologyReport, CriticalFinding
│   PeerReview, DoseRecord, Modality, ImagingProtocol
│
├── STAGES
│   ImagingOrder → ordered → scheduled → in_progress → images_acquired
│               → pending_read → preliminary_read → final_report → addendum → cancelled
│   RadiologyReport → draft → preliminary → final → addended
│   CriticalFinding → detected → communicated → acknowledged → documented
│   PeerReview → assigned → in_progress → completed → discrepancy_escalated
│
├── PACKET × 4
│   ModalityWorklistPacket → DICOM gateway
│   CriticalFindingAlertPacket → clinical, patient portal (exactly_once)
│   FinalReportPacket → clinical, patient portal
│   DoseAlertPacket → governance/ES
│
├── MODULE × 5
│   RISEngine         → STAT/urgent tracking, contrast safety blocks, accession integrity
│   ReportingEngine   → critical finding 60-min rule, report turnaround, impression required
│   PeerReviewEngine  → independence enforcement, RADPEER scoring, concordance tracking
│   DoseEngine        → DRL comparison, 3x alert, cumulative dose, protocol deviation
│   ProtocolEngine    → missing protocol, annual review, premedication protocol
│
├── REASONER × 1
│   radiology_quality_reasoner → weekly: TAT, critical finding compliance, concordance, dose trends
│
├── WORKFLOW × 5
│   schedule_imaging, complete_study, sign_report
│   peer_review_assignment, communicate_critical_finding
│
└── ACTION × 18
    Deterministic: CheckContrastScreening, AssignModalitySlot, EmitModalityWorklist,
    RecordStudyCompletion, RecordDoseMetrics, EvaluateDoseAgainstDRL, AssignRadiologist,
    ValidateReportCompleteness, SignRadiologyReport, CreateCriticalFindingRecord,
    CommunicateCriticalFinding, SelectStudiesForPeerReview, AssignPeerReviewers
    AI (claude-sonnet): GenerateRadiologyReport, IdentifyCriticalFindings,
    SelectImagingProtocol
    AI (claude-haiku): GeneratePeerReviewReport
```

---

## Integration

- → **Clinical**: `FinalReportPacket`, `CriticalFindingAlertPacket`
- → **Patient Portal**: result release after provider acknowledgement
- → **Governance/ES**: `DoseAlertPacket` for high-dose events
- ← **Clinical**: imaging orders via CPOE
- ← **Scheduling**: room and equipment coordination
