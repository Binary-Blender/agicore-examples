# Accelerando Pharmacy

**Two drug safety checks. Two points. One safety net with no gaps.**

> The prescriber checks drug interactions at order entry.  
> The pharmacist checks at dispense.  
> If the patient picked up a new medication between those two events,  
> the pharmacist check catches what the prescriber couldn't have seen.  
> Most pharmacy systems do one check. This one does both.

---

## The PDMP Block

```
RULE controlled_without_pdmp {
  WHEN PDMPContext.controlled_no_pdmp_query == true
  THEN FLAG "controlled_substance_cannot_be_dispensed_without_PDMP_query"
  SEVERITY critical
  PRIORITY 100
}
```

PRIORITY 100. No controlled substance — Schedule II through V — is dispensed without a current PDMP query. "Current" is configurable; the default is 3 days. This fires at dispense even if the prescriber checked PDMP at order entry. Three days is a long time in opioid prescribing.

The PDMP response is evaluated for:

| Flag | What It Catches |
|---|---|
| `doctor_shopping_pattern` | Multiple prescribers detected — PRIORITY 97 |
| `opioid_benzo_combination` | Concurrent opioid + benzodiazepine — PRIORITY 99 |
| `high_mme_threshold_exceeded` | > 90 MME/day (CDC 2022 guideline) — PRIORITY 96 |
| `multiple_pharmacies` | Filling at multiple pharmacies — PRIORITY 97 |

The opioid + benzo rule is the highest after PDMP non-query itself. The combination increases overdose risk 3–4x. The CDC guideline is to avoid concurrent prescribing whenever possible.

---

## E-Prescribing of Controlled Substances

```
RULE controlled_must_be_electronic {
  WHEN ERxContext.controlled_paper_not_electronic == true
  THEN FLAG "controlled_substance_must_use_EPCS_paper_prescription_blocked"
  SEVERITY critical
  PRIORITY 100
}
```

Paper prescriptions for controlled substances are blocked. EPCS (Electronic Prescribing for Controlled Substances) is required for all Schedule II-V. EPCS credentials are tracked — expired credentials block controlled substance prescribing before the prescriber tries to sign.

---

## Formulary Checking — Real Time at the Point of Order

The formulary check runs when the prescription enters the system — before the patient picks up:

1. **Tier** — what is the patient's copay?
2. **PA Required** — does the payer require prior authorization?
3. **Step Therapy** — must a preferred agent be tried first?
4. **Quantity Limit** — does the quantity ordered exceed the formulary limit?
5. **Specialty Drug** — tier 5 routing to specialty pharmacy?

```
RULE pa_required_no_auth {
  WHEN FormularyContext.pa_required_no_auth_on_file == true
  THEN FLAG "prior_authorization_required_initiate_before_dispensing"
  SEVERITY critical
  PRIORITY 93
}
```

If PA is required and none is on file: the prescription routes to the PA workflow automatically. `GeneratePARequest` (AI) generates the clinical criteria narrative for payer submission — not a generic form, a specific narrative matching the payer's criteria for that drug and indication.

---

## The Two-Point Interaction Check

**At order entry (Clinical):** `CheckDrugInteractions` — checks the new medication against all active medications.

**At dispense (Pharmacy):** `CheckInteractionsAtDispense` — checks again. New medications may have been added since the order was written. The pharmacist check is not redundant — it is the safety net for everything that changed.

```
RULE contraindicated_at_dispense {
  WHEN InteractionContext.contraindicated_combination == true
  THEN FLAG "contraindicated_drug_combination_dispense_blocked_pharmacist_contact_prescriber"
  SEVERITY critical
  PRIORITY 100
}
```

Contraindicated combinations block dispense completely. The pharmacist must contact the prescriber. This is not an alert that can be clicked through.

---

## Refill Management

```
RULE controlled_early_refill {
  WHEN RefillContext.controlled_refill_early == true
  THEN FLAG "controlled_substance_early_refill_blocked_state_law_compliance"
  SEVERITY critical
  PRIORITY 100
}

RULE refill_too_soon {
  WHEN RefillContext.refill_too_soon == true
  THEN FLAG "refill_too_soon_more_than_25pct_days_remaining"
  SEVERITY warning
  PRIORITY 85
}
```

Non-controlled refill requests are flagged when more than 25% of days supply remains. Controlled refill requests are blocked based on state law compliance — which is stricter than the 25% rule in most states.

Portal refill requests for controlled substances are blocked by default — configurable via `PREFERENCE refill_request_controlled_blocked`. The patient must call or visit.

---

## Architecture

```
accelerando_pharmacy.agi
│
├── ENTITY × 7
│   Prescription, DispensedMedication, FormularyEntry
│   PriorAuthorization, PDMPQuery, DrugInteractionRecord
│   RefillRequest, PharmacyBenefit
│
├── STAGES
│   Prescription → received → formulary_check → prior_auth_pending / ready_to_fill
│               → in_progress → dispensed / rejected / cancelled
│   PriorAuthorization → submitted → pending → approved / denied → appealed → expired
│   RefillRequest → pending → approved / denied → transmitted
│
├── PACKET × 4
│   PrescriptionReceivedPacket → (inbound from clinical)
│   PriorAuthRequestPacket → billing
│   DispenseConfirmationPacket → clinical, billing
│   PDMPHighRiskPacket → clinical, governance/ES (exactly_once)
│
├── MODULE × 5
│   ERxEngine       → EPCS enforcement, transmission failures, NCPDP acknowledgement
│   FormularyEngine → tier, PA required, step therapy, quantity limit, specialty routing
│   PDMPEngine      → mandatory query, staleness, doctor shopping, MME, opioid+benzo
│   InteractionEngine → contraindicated, major, allergy conflict, duplicate therapy at dispense
│   RefillEngine    → too-soon, no refills, controlled early, PA expiration at refill
│
├── REASONER × 1
│   pharmacy_safety_reasoner → weekly: PDMP patterns, interaction overrides, PA burden, formulary adherence
│
├── WORKFLOW × 5
│   receive_prescription, query_pdmp, fill_prescription
│   initiate_prior_auth, process_refill
│
└── ACTION × 18
    Deterministic: ValidatePrescription, CheckFormulary, QueryPDMP, EvaluatePDMPResponse,
    CheckAllergyAtDispense, CheckInteractionsAtDispense, VerifyPDMPCurrent,
    PharmacistVerification, RecordDispense, TransmitPrescription, SubmitPriorAuth,
    CheckRefillEligibility, FindFormularyAlternatives
    AI (claude-sonnet): GeneratePARequest — clinical narrative for payer submission
```

---

## Seed Data

4 pre-seeded formulary entries demonstrating tier structure:

| Drug | Tier | PA Required | Step Therapy |
|---|---|---|---|
| lisinopril 10mg | 1 (generic) | No | No |
| metformin 500mg | 1 (generic) | No | No |
| atorvastatin 40mg | 2 (preferred brand) | No | No |
| adalimumab 40mg | 5 (specialty) | Yes | Yes |

---

## Connected to the Stack

- ← **Clinical**: receives `PrescriptionPacket` (exactly_once) for every medication order
- → **Clinical**: `DispenseConfirmationPacket` for medication reconciliation
- → **Clinical**: `PDMPHighRiskPacket` for provider awareness and documentation
- → **Billing**: `DispenseConfirmationPacket` for claims
- ← **Patient Portal**: refill requests routed via `RefillRequestPacket`
