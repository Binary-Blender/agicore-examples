# Accelerando Legal

**eDiscovery and legal hygiene for organizations that would like to stop losing cases they created themselves.**

> "Stop putting this stuff in email."  
> "Stop documenting non-compliance."  
>  
> Two rules. Eighty percent of preventable litigation loss. Now encoded.

---

## The Two Layers

Most legal software does one thing: help you respond to litigation after the fact. Document management, hold tracking, production workflows. Necessary, but reactive. You're already in trouble.

Accelerando Legal does two things:

**Layer 1 — Reactive: eDiscovery**  
When litigation happens, manage it. Holds trigger automatically on litigation matters. Custodian notifications go out within 24 hours or the ES flags a violation. Connectors pull from Exchange, Gmail, Slack, OneDrive, SharePoint, and every other Accelerando module. Documents are collected, deduplicated, coded for privilege and responsiveness, Bates numbered, and produced — with a full privilege log and an immutable chain of custody record.

**Layer 2 — Proactive: Legal Hygiene**  
Stop the documents from being created in the first place. AI runs once at build time (`GenerateLegalHygienePatterns`) to generate PATTERN declarations from your industry's liability landscape. At runtime, deterministic pattern matching scans every communication across every connected system. Nothing hallucinates at scan time. Nothing is guessing. The patterns either match or they don't.

The patterns that match are the ones that become Plaintiff's Exhibit 1.

---

## The Two Pet Peeves

### "Stop Documenting Non-Compliance"

Every year, companies lose cases they would have won — or paid nothing on — because someone wrote a memo. Not a smoking-gun memo. A routine one. An email from the compliance officer to the VP of Operations: *"As we discussed, we are aware that our current safety procedure doesn't meet the updated standard. We've flagged this internally."*

Two years later, that email is produced in discovery. Plaintiff's counsel puts it on a slide. Case over.

The pattern:
```
PATTERN known_noncompliance_language {
  MATCH    "we are aware that|despite our policy|we have known since|
            although not standard practice|not in compliance with our own|
            we acknowledged|known deficiency|contrary to our policy"
  RESPOND  "LEGAL HYGIENE ALERT: Language documenting known non-compliance detected.
            Legal review required before this document is forwarded or filed."
  SCORE    legal_risk_score -= 20
  ASSERT   HygieneContext.documents_known_noncompliance = true
}

RULE self_incrimination_critical {
  WHEN HygieneContext.documents_known_noncompliance == true
  THEN FLAG "documented_noncompliance_legal_review_required"
  SEVERITY critical
  PRIORITY 100
}
```

The flag fires. Legal gets a notification. The document gets reviewed before it proliferates across six more email threads and twelve more custodians.

### "Stop Putting This Stuff in Email"

When people anticipate litigation and discuss it in unprotected channels, they hand opposing counsel a roadmap. Attorney-client privilege requires an attorney. A Slack message to your VP of Operations about your "legal exposure on the Hendricks matter" doesn't have one.

```
PATTERN litigation_anticipation_language {
  MATCH    "if this goes to court|when we get sued|legal exposure|
            litigation risk|our lawyers will|potential liability|
            plaintiff will|this will look bad|if deposed|settlement exposure"
  RESPOND  "LEGAL HYGIENE ALERT: Litigation anticipation language in unprotected channel.
            Attorney-client privilege requires an attorney in the communication.
            This message may not be protected."
  SCORE    legal_risk_score -= 15
  ASSERT   HygieneContext.litigation_anticipation_in_email = true
}
```

The worst one in the library:

```
PATTERN cover_up_language {
  MATCH    "don't put this in writing|don't document this|off the record|
            delete this email|destroy these|no paper trail|
            keep this between us|call me don't email"
  RESPOND  "LEGAL HYGIENE ALERT: Language suggesting suppression of documentation.
            This communication is itself now discoverable evidence.
            Immediate legal counsel notification recommended."
  SCORE    legal_risk_score -= 25
  ASSERT   HygieneContext.cover_up_language_detected = true
}
```

Attempting to suppress evidence is worse than the underlying issue. Courts take a very dim view. So does the jury.

---

## Five Hygiene Patterns

| Pattern | What It Catches | Risk Score Impact |
|---|---|---|
| `known_noncompliance_language` | "We are aware that... / despite our policy... / we have known since..." | −20 |
| `litigation_anticipation_language` | "If this goes to court / legal exposure / when we get sued..." | −15 |
| `cover_up_language` | "Don't put this in writing / delete this / no paper trail..." | −25 |
| `privilege_waiver_language` | Attorney advice forwarded to non-privileged parties | −20 |
| `hr_liability_language` | Personnel discussions outside formal HR channels | −15 |

`GenerateLegalHygienePatterns` generates industry-specific additions at build time. A healthcare deployment adds HIPAA-specific patterns. A financial services deployment adds SEC/FINRA patterns. A government contractor deployment adds FAR/False Claims Act patterns. The pattern library grows with the industry.

---

## The Legal Risk Score

```
SCORE legal_risk_score {
  INITIAL  100
  MIN        0
  MAX      100
  DECAY      0 PER day          // risk doesn't decrease without action
  THRESHOLD elevated AT 75 THEN FLAG "legal_risk_elevated"
  THRESHOLD high     AT 50 THEN FLAG "legal_risk_high_notify_counsel"
  THRESHOLD critical AT 25 THEN FLAG "legal_risk_critical_immediate_action"
}
```

No decay. Risk doesn't expire. The score falls when patterns are detected and rises when flags are reviewed and resolved. At 50, outside counsel gets notified. At 25, immediate action is required. The score is the organization's running legal hygiene record.

---

## Connector Architecture — Every Source, One System

Legal discovery touches every data source the organization has. This module connects them all.

| Connector | Source | Sync Interval | What It Captures |
|---|---|---|---|
| Exchange | Microsoft 365 / Graph API | 24h (configurable) | Email, calendar, contacts |
| Gmail | Google Workspace / Gmail API | 24h | Email, labels, threads |
| Slack | Slack API | 24h | Messages, DMs, channels, reactions |
| OneDrive | Microsoft 365 / Graph API | 24h | Files, SharePoint documents |
| Google Drive | Drive API | 24h | Files, shared documents |
| ERP | Accelerando internal | real-time | Contracts, invoices, HR, billing records |

Every message pulled from every connector runs through the hygiene PATTERN matching before it lands in the review queue. A PATTERN match creates a `LegalHygieneFlag` record and emits a signed `HygieneAlertPacket` to legal. This happens at ingest time — before the document is reviewed, before a hold is triggered, before anyone asks for it.

Connector staleness is enforced:
```
RULE exchange_sync_overdue {
  WHEN ConnectorContext.exchange_connected == true
  AND  ConnectorContext.exchange_sync_stale == true
  THEN FLAG "exchange_connector_sync_overdue"
  SEVERITY warning
  PRIORITY 70
}
```

If a connector fails during a legal hold, `LegalHoldContext.preservation_failed = true` and the ES escalates to PRIORITY 100 immediately. You don't find out during production that three months of email are missing.

---

## Legal Hold Workflow

```
trigger_legal_hold:
  assess_matter          → AssessLegalHoldRequirements  (matter type? custodians? data sources?)
  trigger_hold           → TriggerLegalHold             (create hold, suspend deletion)
  notify_custodians      → NotifyCustodians             (emit LegalHoldNoticePacket per custodian)
  sync_connectors        → SyncAllConnectors            (snapshot all in-scope data)
  preserve_data          → PreserveCustodianData        (preserve per custodian per connector)
  log                    → LogLegalEvent                (immutable chain of custody entry)
```

Hold notice generation is an AI action. `GenerateHoldNotice` takes the matter description and custodian profile and produces a formal, clear hold notice — professional enough that the custodian understands what's required without triggering unnecessary alarm.

Acknowledgment tracking:
- Notices issued with 48-hour acknowledgment deadline (configurable)
- `LegalHoldContext.hold_acknowledgment_overdue = true` after deadline → PRIORITY 80 FLAG → escalation
- Hold is never considered complete until all custodians have acknowledged

Deletion suspension is not advisory:
```
RULE block_deletion_during_hold {
  WHEN LegalHoldContext.deletion_attempt_during_hold == true
  THEN FLAG "deletion_blocked_active_legal_hold"
  SEVERITY critical
  PRIORITY 100
}
```

Document deletion during a legal hold is spoliation. The ES treats it as such.

---

## Document Review and Production

Collection → deduplication → review assignment → privilege coding → Bates numbering → privilege log → production.

Every step is logged. Chain of custody is maintained from source document through produced page.

```
RULE privilege_log_before_production {
  WHEN ProductionContext.production_set_ready == true
  AND  ProductionContext.privilege_log_complete == false
  THEN FLAG "privilege_log_must_be_complete_before_production"
  SEVERITY critical
  PRIORITY 100
}
```

You cannot produce without a privilege log. The ES blocks it. Producing without a privilege log waives privilege on everything in the set — a mistake that cannot be undone.

Production deadline enforcement:
- Warning at 14 days remaining + review incomplete → FLAG
- Critical at 5 days remaining → PRIORITY 100 FLAG
- `review_velocity` SCORE decays 2 points per day → flags slow/stalled reviews before deadlines are missed

---

## The Legal Advisor Voice

`legal_risk_reasoner` runs weekly. It is a REASONER — AI at scheduled time, reviewing accumulated flags and producing a plain-English briefing:

```
REASONER legal_risk_reasoner {
  SCHEDULE "weekly"
  INPUT    LegalHygieneFlag, LegalMatter, HygieneContext
  PROMPT   "You are a legal risk advisor reviewing flagged communications from the past 7 days.
            For each flag type: what risk it creates, which litigation scenario it feeds,
            what the organization can do about it today. Tone: direct, non-alarmist, but clear.
            Do not give legal advice. Give legal awareness.
            If you see documented non-compliance language, say so plainly.
            If you see cover-up language, say that attempting to suppress evidence
            is worse than the underlying issue."
  OUTPUT   LegalHygieneFlag
}
```

The output is a weekly advisory memo: executive summary, matter status, hygiene flag summary, recommended actions. Not a lawyer. A very well-organized advisor who has seen how these patterns end.

---

## Architecture

```
accelerando_legal.agi
│
├── ENTITY × 9
│   LegalMatter           → matter registry, type, status, risk level
│   LegalHold             → preservation holds tied to matters
│   HoldCustodian         → per-person hold tracking, notification, acknowledgment
│   ExternalConnector     → connector config for Exchange, Gmail, Slack, OneDrive, GDrive, ERP
│   DocumentCollection    → collection runs with search terms and date ranges
│   ReviewDocument        → individual documents with responsiveness and privilege coding
│   PrivilegeLogEntry     → privilege log per produced document
│   ProductionSet         → production batches with Bates ranges
│   LegalHygieneFlag      → flagged communications from PATTERN matching
│   RetentionPolicy       → retention schedule by document category
│
├── STAGES
│   LegalMatter.status    → intake → active → hold_triggered → in_review → production → closed
│   ReviewDocument.status → collected → processed → assigned → reviewed → produced
│   ProductionSet.status  → building → bates_applied → privilege_logged → produced → confirmed
│
├── PACKET × 6            → typed schemas for all connector data and outbound notifications
│   ExchangeMailPacket, GmailPacket, SlackMessagePacket, CloudFilePacket
│   LegalHoldNoticePacket, HygieneAlertPacket
│
├── CHANNEL × 8           → inbound per connector + outbound for notices and alerts
│
├── MODULE × 6
│   LegalHoldEngine       → LegalHoldContext FACT, hold lifecycle, LegalHoldFlow STATE
│   LegalHygieneAdvisor   → HygieneContext FACT, 5 PATTERNs, 5 RULEs, legal_risk_score SCORE
│   DocumentReviewEngine  → ReviewContext FACT, deadline enforcement, review_velocity SCORE
│   ProductionEngine      → ProductionContext FACT, pre-production gates, ProductionFlow STATE
│   ConnectorEngine       → ConnectorContext FACT, staleness detection per connector
│   RetentionEngine       → RetentionContext FACT, deletion suspension, policy gap detection
│   RiskIntelligence      → RiskContext FACT, report trigger RULEs
│
├── REASONER × 1
│   legal_risk_reasoner   → weekly batch, hygiene flags → plain-English risk advisory memo
│
├── WORKFLOW × 6
│   trigger_legal_hold           → 6 steps: assess → trigger → notify → sync → preserve → log
│   collect_and_process          → 4 steps: sync → collect → assign → log
│   privilege_review_and_production → 4 steps: privilege_log → bates → produce → log
│   release_hold                 → 2 steps: release → log
│   legal_hygiene_scan           → 7 steps: scan all connectors → analyze → log
│   weekly_legal_risk_assessment → 4 steps: scan → analyze → report → log
│
├── ACTION × 20
│   GenerateLegalHygienePatterns → AI: industry + regulatory context → PATTERN DSL
│   GenerateHoldNotice           → AI: matter + custodian → formal hold notice
│   AnalyzeCommunicationRisk     → AI: flags → risk assessment + behavioral recommendations
│   GenerateLegalRiskReport      → AI: matters + flags + trends → weekly advisory memo
│   All others                   → deterministic: sync, collect, assign, Bates, produce, release
│
├── VIEW × 7
│   LegalMatterDashboard, HoldManagement, DocumentReview
│   LegalHygieneMonitor, ConnectorStatus, ProductionManager, RetentionPolicies
│
└── PREFERENCE × 6
    hygiene_scan_interval_hours, production_deadline_warning_days,
    connector_sync_stale_hours, legal_risk_notify_threshold,
    hold_ack_deadline_hours, retention_expiry_warning_days
```

---

## Default Retention Policies

Six policies pre-loaded. Override or extend per regulatory requirement.

| Category | Retention | Basis |
|---|---|---|
| Email Correspondence | 7 years | General business |
| Contracts and Agreements | 10 years | Statute of limitations |
| Financial Records | 7 years | IRS |
| HR and Personnel | 7 years | EEOC |
| Compliance Documentation | 10 years | Regulatory |
| General Business Records | 5 years | Internal |

All retention policy enforcement is suspended when a legal hold is active on the document's custodian. Retention never deletes a document that is subject to a hold.

---

## The Full Accelerando Stack

```
accelerando_erp.agi          → business data (32 entities, 32 actions)
accelerando_billing.agi      → medical billing (full claim lifecycle, self-updating rules)
accelerando_legal.agi        → eDiscovery and legal hygiene (this app)
accelerando_interchange.agi  → standard data interchange (HL7, FHIR, X12, EDIFACT, RosettaNet)
accelerando_config.agi       → self-configuration (13 templates, 6 advisory modules)
accelerando_chatbot.agi      → customer service (deterministic, cannot hallucinate)
accelerando_eliza.agi        → operator interface (20 workflows, macro executor)
accelerando_es.agi           → governance (34 rules, 6 policy modules)
accelerando_oie.agi          → organizational intelligence (AI reasoning, retrospective)
```

Nine `.agi` files. The legal layer governs every other module's data — when a hold is triggered, the ERP records, the billing records, the interchange messages, and the chat logs are all preserved and discoverable. Every communication that passes through Slack, email, or any connected system runs through hygiene pattern matching before it becomes the document someone wishes they'd never written.

The organizations that lose cases they should have won almost never lose on the underlying facts. They lose on the paper trail they created. This system reads the paper trail before the plaintiff does.
