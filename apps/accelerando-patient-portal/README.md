# Accelerando Patient Portal

**Results when the provider has reviewed them. Not before.**

> The most common patient complaint about portals is not a technical failure.  
> It's finding their own cancer diagnosis in an automated result release  
> before the doctor called them.  
> This system prevents that. Every time.

---

## The Result Release Rules

Two hard blocks on result release. PRIORITY 100, both:

```
RULE critical_result_never_auto_release {
  WHEN ResultsContext.critical_result_auto_release_attempted == true
  THEN FLAG "critical_result_auto_release_blocked_provider_must_release_manually"
  SEVERITY critical
  PRIORITY 100
}

RULE abnormal_result_requires_review {
  WHEN ResultsContext.abnormal_result_unreview_release_attempted == true
  THEN FLAG "abnormal_result_release_blocked_pending_provider_review"
  SEVERITY critical
  PRIORITY 100
}
```

Critical results: never auto-release. The provider must manually release after communication.

Abnormal results: held until provider review is on record. The provider reviews, adds a note if warranted, then releases.

Normal results: released automatically after provider acknowledgement. The patient gets a notification. The wait time for a normal result is minutes to hours — not days.

The `ReleaseResult` action in the clinical system checks review status before writing to the portal. It cannot be bypassed. The portal cannot pull results — results are pushed by the clinical system after the safety check.

---

## Messaging SLAs — Enforced

```
RULE urgent_message_unresponded {
  WHEN MessagingContext.urgent_message_unresponded_4h == true
  THEN FLAG "urgent_portal_message_unresponded_4h_escalate_to_on_call"
  SEVERITY critical
  PRIORITY 92
}

RULE message_response_sla {
  WHEN MessagingContext.message_unresponded_2_business_days == true
  THEN FLAG "portal_message_unresponded_2_business_days_sla_breach"
  SEVERITY warning
  PRIORITY 80
}
```

Patients send messages with urgency levels. Urgent messages flag to on-call after 4 hours unresponded. Standard messages flag at 2 business days. After-hours urgent messages route to the on-call provider immediately.

Message routing is automatic — `RouteMessageToProvider` categorizes the message by subject and care team membership, routes to the correct inbox, and notifies the provider. The patient doesn't need to know which provider to address.

---

## Access Control — NIST IAL2

```
RULE identity_not_verified {
  WHEN AccessContext.account_not_identity_verified == true
  THEN FLAG "portal_account_identity_not_verified_limit_access_to_non_clinical"
  SEVERITY warning
  PRIORITY 85
}

RULE account_locked {
  WHEN AccessContext.account_locked_failed_logins == true
  THEN FLAG "portal_account_locked_failed_login_attempts_identity_verify_before_unlock"
  SEVERITY critical
  PRIORITY 95
}
```

Identity verification is NIST 800-63 IAL2. Until verified: non-clinical access only (appointments, messaging with no sensitive clinical data). Full clinical access — results, records, medications — requires verified identity.

Accounts lock after 5 failed login attempts. Unlock requires identity re-verification, not just a password reset. MFA is on by default. Disabling MFA generates a security flag.

---

## Controlled Substance Refills — Hard Block

```
RULE controlled_via_portal_blocked {
  WHEN RefillPortalContext.refill_for_controlled_via_portal == true
  THEN FLAG "controlled_substance_refill_via_portal_blocked_phone_or_office_visit_required"
  SEVERITY critical
  PRIORITY 100
}
```

No controlled substance refills via portal. The patient must call or visit. This is configurable via `PREFERENCE refill_request_controlled_blocked` but defaults to true.

Non-controlled refill requests route to clinical for review. `ValidateRefillRequest` checks: medication is active, not controlled, pharmacy on file, refills remaining. Clinical reviews within 48 hours. Approved: pharmacy receives a new NCPDP SCRIPT transmission.

---

## Health Summary — Plain Language

```
ACTION GeneratePatientHealthSummary {
  AI_MODEL "claude-sonnet-4-6"
  INPUT {
    preferred_language: string  DEFAULT "English"
    reading_level:      string  DEFAULT "8th_grade"
    health_data:        string  REQUIRED
  }
}
```

`GeneratePatientHealthSummary` produces a patient-readable summary at 8th grade reading level (configurable). Problems described in plain language, not ICD-10 codes. Medications listed with plain-language instructions. Allergies and immunizations summarized. The same data from `CompileChartData` that generates a CCD also generates a health summary that a patient can actually read.

The summary is available in the portal, downloadable as PDF, and available in the patient's preferred language.

---

## Proxy Access — Documented and Expiring

Parents for minor children. Authorized representatives for adults who need assistance. The proxy access lifecycle:

```
ValidateProxyRelationship → legal documentation confirmed
       ↓
CreateProxyPortalAccount → proxy account linked to patient
       ↓
RecordProxyConsentGrant → patient consent on file
       ↓
Proxy access expires per PREFERENCE (default: 1 year)
       ↓
RULE proxy_expired → FLAG "proxy_access_expired_renew_or_revoke"
```

```
RULE minor_without_proxy {
  WHEN AccessContext.minor_patient_direct_access == true
  THEN FLAG "minor_patient_direct_portal_access_review_state_law_requirements"
  SEVERITY warning
  PRIORITY 80
}
```

Minor patients with direct access (not via parent proxy) trigger a state law review flag. The rules for minor portal access vary by state — the system flags it for compliance review rather than blocking outright.

---

## Architecture

```
accelerando_patient_portal.agi
│
├── ENTITY × 7
│   PortalAccount, SecureMessage, ResultRelease
│   AppointmentRequest, RefillRequest, HealthSummary
│   ConsentRecord, HealthGoal
│
├── STAGES
│   SecureMessage → sent → read → replied / closed / escalated
│   ResultRelease → pending_review → approved → released → viewed / held
│   AppointmentRequest → submitted → processing → confirmed / denied / cancelled
│   RefillRequest → submitted → pending_review → approved / denied → transmitted
│
├── PACKET × 4
│   PortalMessagePacket → clinical
│   RefillRequestPacket → pharmacy (exactly_once)
│   AppointmentRequestPacket → scheduling
│   ResultViewedPacket → clinical
│
├── MODULE × 4
│   ResultsEngine  → critical never-release (PRIORITY 100), abnormal review required (PRIORITY 100), 7-day hold flag
│   MessagingEngine → urgent 4h escalation (PRIORITY 92), 2-day SLA breach, after-hours routing
│   AccessEngine   → IAL2 verification, account lock, proxy expiration, MFA, minor access
│   RefillEngine   → controlled substance block (PRIORITY 100), 48h review SLA
│
├── REASONER × 1
│   portal_engagement_reasoner → weekly: adoption metrics, message volume, result viewing rates
│
├── WORKFLOW × 7
│   enroll_patient, send_message, release_result
│   submit_refill_request, self_schedule_appointment
│   generate_health_summary, manage_proxy_access
│
└── ACTION × 22
    Deterministic: CreatePortalAccount, InitiateIdentityVerification, ValidatePortalAccountAccess,
    CreateSecureMessage, RouteMessageToProvider, CheckResultReleaseEligibility,
    CreateResultReleaseRecord, NotifyPatientOfResult, ValidateRefillRequest,
    CreatePortalRefillRequest, ValidateSelfScheduleEligibility, CompileHealthSummaryData,
    ValidateProxyRelationship, RevokeProxyAccess
    AI (claude-sonnet): GeneratePatientHealthSummary — plain language, preferred language, reading level
```

---

## Connected to the Stack

The patient portal is the patient-facing layer for every clinical system:

- ← **Clinical**: result releases, CCD delivery, care gap alerts, critical finding notifications
- ← **Scheduling**: appointment confirmations, cancellations, recall notifications
- ← **Population Health**: care gap outreach, care plan access
- → **Clinical**: portal messages, result viewed acknowledgements
- → **Pharmacy**: refill requests
- → **Scheduling**: appointment requests
