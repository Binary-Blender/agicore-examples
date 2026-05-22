# Accelerando Quality Management System

**ISO 9001:2015. Every clause enforced. No theater.**

> A CAPA with no effectiveness check is theater.  
> An audit with no corrective actions is theater.  
> A management review with no decisions is theater.  
> This system enforces the loop.

---

## What Most QMS Implementations Miss

ISO 9001 tells you *what* you must have. It doesn't enforce *whether it works*.

Most QMS implementations produce: a document library, a CAPA register, and annual audit anxiety. The CAPAs accumulate. The effectiveness checks never happen. The management review produces minutes but no decisions. The auditors find the same findings every year because the corrective actions from last year weren't verified.

The QMS loop is: nonconformance → root cause → corrective action → effectiveness check → closure. Every link matters. The loop only works if every link is enforced.

This system enforces every link. The ES tracks effectiveness check deadlines. It escalates overdue CAPAs. It requires management review inputs and outputs per Clause 9.3.2 and 9.3.3. It blocks management review closure without documented outputs. It detects when the same root cause appears for the second time and triggers a PI CoE referral, because if the corrective action didn't prevent recurrence, it wasn't corrective — it was reactive.

---

## The Closed Loop

```
Nonconformance detected
       ↓
NCR created (Clause 8.7) — objective evidence required, not just description
       ↓
Root cause analysis — 5-Why, Ishikawa, or Fault Tree
       ↓
CAPA initiated (Clause 10.2) — due date assigned, owner notified
       ↓
Implementation
       ↓
Effectiveness check at 90 days — did the root cause recur?
       ↓
If effective → CAPA closed
If ineffective → CAPAContext.ineffective_capas += 1 → new CAPA triggered,
                 root cause revisit required, PI CoE referral if systemic
```

The effectiveness check is not optional. The ES tracks effectiveness_due for every CAPA. At the due date:
- Not verified → FLAG: `capa_effectiveness_check_overdue_capa_at_risk`
- Verified ineffective → FLAG: `ineffective_capa_root_cause_revisit_required` + new CAPA

The second CAPA for the same root cause triggers `systemic_issue_detected = true` → `ReferToPICoE`. Because if the same problem recurred after a corrective action, this is a systemic issue that requires an improvement project, not another CAPA.

---

## ISO 9001:2015 Clause Coverage

| Clause | Subject | Module |
|---|---|---|
| 4.1–4.4 | Context, interested parties, QMS scope | RiskOpportunity entity |
| 5.1–5.3 | Leadership, quality policy, roles | ManagementReviewEngine |
| 6.1–6.2 | Risks/opportunities, quality objectives | QualityObjective, RiskOpportunity |
| 7.1.5 | Calibration and measurement | CalibrationEngine |
| 7.5 | Documented information | DocumentControlEngine |
| 8.2.1 | Customer communication | CustomerFeedbackEngine |
| 8.4 | External providers (suppliers) | SupplierQualityEngine |
| 8.7 | Control of nonconforming outputs | NonConformanceEngine |
| 9.1.2 | Customer satisfaction | CustomerFeedbackEngine |
| 9.2 | Internal audit | AuditEngine |
| 9.3 | Management review | ManagementReviewEngine |
| 10.2 | Nonconformance and corrective action | CorrectiveActionEngine |

---

## Document Control — Obsolete Documents Kill Products

The most common audit finding in document control is also the most preventable: obsolete documents in use on the shop floor. Someone is following revision 3 of a work instruction while revision 5 is in the document system.

```
RULE obsolete_document_in_use {
  WHEN DocumentContext.obsolete_documents_in_use == true
  THEN FLAG "obsolete_document_in_use_withdraw_immediately"
  SEVERITY critical
  PRIORITY 100
}
```

PRIORITY 100. The system does not treat document control violations as minor inconveniences. The next nonconformance report tracing back to an obsolete procedure is a CAPA, a customer complaint, and potentially a product recall.

Document review dates are tracked. At review_frequency_days (default 365), the document goes overdue:

```
RULE document_review_overdue {
  WHEN DocumentContext.documents_overdue_for_review > 0
  THEN FLAG "document_review_overdue_schedule_review"
  SEVERITY warning
  PRIORITY 75
}
```

13 core ISO 9001 required documents are pre-seeded: Quality Manual, and 12 procedures and forms covering all mandatory clause requirements. The document register is a starting point, not a ceiling.

---

## Calibration — Out-of-Tolerance Equipment in Use

```
RULE out_of_tolerance_equipment_in_use {
  WHEN CalibrationContext.out_of_tolerance_equipment == true
  THEN FLAG "out_of_tolerance_equipment_in_use_product_validity_at_risk"
  SEVERITY critical
  PRIORITY 100
}

RULE retroactive_measurement_review {
  WHEN CalibrationContext.measurements_with_expired_cal == true
  THEN FLAG "measurements_taken_with_expired_calibration_review_required"
  SEVERITY critical
  PRIORITY 95
}
```

The second rule is the expensive one. When equipment goes out of calibration, every measurement taken with that equipment since the last known-good calibration is now suspect. The system tracks which measurements were taken with which equipment and flags retroactive review when a calibration gap is discovered. This is Clause 7.1.5.2(d) — and most QMS implementations don't enforce it.

---

## Internal Audit — The Independence Rule

```
RULE auditor_independence {
  WHEN AuditContext.auditor_independence_violated == true
  THEN FLAG "auditor_independence_violation_iso9001_clause_9.2.2"
  SEVERITY critical
  PRIORITY 100
}
```

ISO 9001 Clause 9.2.2(b): auditors shall not audit their own work. This is the most commonly violated internal audit requirement, usually because of resource constraints in smaller organizations. The system tracks auditor assignments against area ownership and flags the conflict before the audit happens, not during the external audit when the registrar notices.

The audit program must cover all ISO 9001 clauses annually:

```
RULE not_all_clauses_covered {
  WHEN AuditContext.all_clauses_covered_ytd == false
  THEN FLAG "audit_program_gap_not_all_iso_clauses_covered_this_year"
  SEVERITY warning
  PRIORITY 80
}
```

`ScheduleInternalAuditProgram` generates the annual audit schedule with clause coverage mapped to areas, independence requirements flagged, and high-risk areas (prior major findings) scheduled for more frequent auditing.

`GenerateAuditReport` produces a report per ISO 19011 — complete with clause references, objective evidence citations, and positive observations. *"Not doing X"* is not a finding. *"Record 12345 does not contain required field Y, per procedure QP-003 Rev 4 Clause 4.2"* is a finding.

---

## Management Review — Decisions, Not Minutes

Management review is the most commonly faked clause in ISO 9001. The organization schedules the meeting, distributes the agenda, files the minutes, and nothing changes. The registrar sees a management review record; the QMS sees no improvement.

Required inputs per Clause 9.3.2 are tracked as boolean flags in `ReviewContext`. Required outputs per Clause 9.3.3 are also tracked. The management review cannot be closed without both documented.

```
RULE required_outputs_missing {
  WHEN ReviewContext.required_outputs_documented == false
  THEN FLAG "management_review_outputs_incomplete_iso9001_9.3.3"
  SEVERITY critical
  PRIORITY 90
}
```

`GenerateManagementReviewPackage` prepares the input document from live QMS data: NCR trends, CAPA effectiveness rates, customer satisfaction scores, audit finding patterns, quality objective performance, and risk register status. It also drafts output decisions based on the analysis — because a management review that doesn't produce decisions is a compliance exercise, not a management tool.

---

## Supplier Quality — Approved Supplier List Enforcement

```
RULE unapproved_supplier {
  WHEN SupplierContext.unapproved_suppliers_in_use == true
  THEN FLAG "unapproved_supplier_in_use_iso9001_clause_8.4"
  SEVERITY critical
  PRIORITY 100
}
```

The approved supplier list is a control. Using an unapproved supplier — for any reason, including "they were the only one who had it in stock" — is a nonconformance. PRIORITY 100. The NCR writes itself.

Supplier performance scores (quality + delivery) are tracked. Below-threshold performance triggers a development plan requirement. Suspended critical suppliers trigger a supply continuity flag.

---

## Connected to the PI CoE

When a CAPA has recurring root cause, the QMS escalates to the PI CoE:

```
RULE systemic_issue_pi_coe_referral {
  WHEN CAPAContext.systemic_issue_detected == true
  THEN ReferToPICoE
  SEVERITY warning
  PRIORITY 85
}
```

`ReferToPICoE` emits a `CAPAToPICoEPacket` with the problem description, root cause, and recurrence count. The PI CoE receives it and can trigger a Kaizen event or DMAIC project. The QMS captures the improvement results as evidence of the corrective action's effectiveness.

The loop: **QMS finds it → PI CoE fixes it permanently → QMS verifies the fix held.**

---

## Architecture

```
accelerando_qms.agi
│
├── ENTITY × 13
│   QMSDocument, DocumentRevision
│   NonConformance, CorrectiveAction
│   InternalAudit, AuditFinding
│   ManagementReview, ManagementReviewAction
│   CustomerComplaint, SupplierAssessment
│   CalibrationRecord, QualityObjective, RiskOpportunity
│
├── STAGES
│   NonConformance    → open → assigned → root_cause → corrective_action → verification → closed
│   CorrectiveAction  → open → in_progress → implemented → effectiveness_check → closed / ineffective
│   InternalAudit     → planned → active → findings_issued → corrective_actions → closed
│   QMSDocument       → draft → in_review → approved → obsolete
│   CustomerComplaint → open → acknowledged → investigating → resolved → closed
│
├── PACKET × 3
│   CAPAToPICoEPacket      → systemic issues to PI CoE for improvement projects
│   NCRFromPICoEPacket     → process defect patterns from PI CoE
│   AuditReadinessPacket   → pre-audit readiness score to leadership
│
├── CHANNEL × 3
│   capa_to_pi_coe, ncr_from_pi_coe, audit_readiness_outbound
│
├── MODULE × 8 (ISO 9001 clause map)
│   DocumentControlEngine    → Clause 7.5: document lifecycle, obsolete document detection
│   NonConformanceEngine     → Clauses 8.7, 10.2: NCR management, recurring NC detection
│   CorrectiveActionEngine   → Clause 10.2: CAPA lifecycle, effectiveness tracking, PI CoE referral
│   AuditEngine              → Clause 9.2: audit program, independence, clause coverage
│   ManagementReviewEngine   → Clause 9.3: review scheduling, required inputs/outputs enforcement
│   SupplierQualityEngine    → Clause 8.4: approved supplier list, performance scoring
│   CalibrationEngine        → Clause 7.1.5: equipment calibration, retroactive review
│   CustomerFeedbackEngine   → Clauses 8.2.1, 9.1.2: complaint handling, customer_satisfaction SCORE
│
├── REASONER × 1
│   quality_intelligence_reasoner → monthly: NCR trends, CAPA effectiveness rates, complaint
│                                   patterns, quality objective performance → management review input
│
├── WORKFLOW × 7
│   ncr_to_capa, capa_effectiveness_check, internal_audit, management_review,
│   handle_customer_complaint, annual_audit_planning, pre_audit_readiness
│
├── ACTION × 8
│   InitiateCorrectiveAction       → deterministic: source → CAPA with deadline + effectiveness date
│   ReferToPICoE                   → deterministic: systemic CAPA → CAPAToPICoEPacket
│   VerifyCAPAEffectiveness        → deterministic: recurrence check → effective/ineffective closure
│   GenerateNCRReport              → AI: NCR data → ISO-compliant NCR document
│   GenerateAuditReport            → AI: audit data → ISO 19011 audit report with evidence
│   ScheduleInternalAuditProgram   → AI: clauses + areas → annual audit schedule
│   AssessAuditReadiness           → deterministic: QMS state → readiness score + report
│   GenerateManagementReviewPackage → AI: QMS data → Clause 9.3.2 input + 9.3.3 draft outputs
│
└── SEED DATA
    13 core ISO 9001 required documents: Quality Manual (QM-001), 8 required procedures,
    4 required forms — pre-populated with correct clause references and document numbers
```

---

## The Full Accelerando Stack

```
accelerando_erp.agi          → business data
accelerando_billing.agi      → medical billing
accelerando_legal.agi        → eDiscovery and legal hygiene
accelerando_lms.agi          → compliance training
accelerando_pi_coe.agi       → process improvement (TPS, Six Sigma, Kaizen, anti-backslide)
accelerando_qms.agi          → quality management system (ISO 9001:2015, this app)
accelerando_interchange.agi  → standard data interchange
accelerando_config.agi       → self-configuration
accelerando_chatbot.agi      → customer service
accelerando_eliza.agi        → operator interface
accelerando_es.agi           → governance
accelerando_oie.agi          → organizational intelligence
```

The QMS and PI CoE are the quality backbone of the manufacturing organization. The QMS captures every nonconformance, drives it to root cause, and forces corrective action with verified effectiveness. The PI CoE takes the systemic problems the QMS surfaces and eliminates them permanently. Neither works as well without the other.
