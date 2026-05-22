# Accelerando Interchange

**Standard data interchange for every industry Accelerando serves.**

> The manufacturing interfaces you built in the 90s? EDI X12.  
> The HL7 interfaces you built in healthcare? Same X12 envelope, different transaction sets.  
> One standard. Two industries. One module.

---

## The Insight

Standard interchange formats are *already deterministic*. The HL7 v2.x spec says exactly which segments are required in an ADT^A01. The X12 spec says exactly which segments are required in an 850. Every validation rule is published. Every field requirement is documented. Encoding them as Agicore RULEs is the obvious thing — and nobody has done it.

Every inbound message follows the same path:

```
External system → parse → validate (published spec as RULEs) → transform → ERP entity → acknowledge
```

Every error has a name. Every missing segment is a named flag. `hl7_missing_pid_segment`. `x12_850_missing_beg_segment`. `edifact_missing_bgm_segment`. Not "parsing failed" — the exact field, the exact reason, the exact spec reference.

---

## Five Interchange Standards

### HL7 v2.x
The pipe-delimited message format that runs every hospital interface engine in North America. Sent over MLLP (TCP) or HTTP.

```
MSH|^~\&|VERTEX_EHR|VERTEX_MAIN|ACCELERANDO|ACME|20260520103000||ADT^A01|MSG001|P|2.5
PID|1||PAT-12345^^^VERTEX||Chen^Alice||19850315|F|||123 Main St^^Portland^OR^97201
PV1|1|I|ICU^101^A|E|||DR12345^Smith^John|||MED|||||||V001|...
```

**Message types processed:**
| Message | Event | Maps to |
|---|---|---|
| ADT^A01 | Patient Admit | Customer + Visit record |
| ADT^A03 | Patient Discharge | Visit close, trigger billing |
| ADT^A08 | Demographics Update | Customer record merge |
| ORM^O01 | Order | ServiceRequest |
| ORU^R01 | Lab/Observation Result | ServiceTicket update |

**Required segment validation (encoded as RULEs):**
- `hl7_pid_required` — ADT without PID → PRIORITY 100 flag
- `hl7_pv1_required_for_adt` — A01 without PV1 → PRIORITY 100 flag
- `hl7_patient_id_required` — any message, missing patient ID → PRIORITY 100 flag
- `hl7_obr_required_for_orm` — ORM without OBR → PRIORITY 100 flag
- `hl7_discard_test_messages` — P vs T in MSH-11 → silently drop, log

### HL7 FHIR R4
The REST-based successor to HL7 v2.x. JSON over HTTPS. Every resource is a typed endpoint.

```
FHIR Patient → Accelerando Customer
FHIR Encounter → ServiceTicket (with clinical metadata)
FHIR Claim → Invoice (with ICD-10/CPT coding)
FHIR Coverage → Contract (subscriber + payer)
FHIR Observation → result stored against patient record
```

**Validation rules:** `fhir_identifier_required`, `fhir_subject_required_for_clinical`, `fhir_r4_version_required`

### ANSI X12 EDI
The workhorse of North American business interchange. Same ISA/GS envelope for supply chain and healthcare — only the transaction set number changes.

**Supply chain transaction sets:**
| Set | Name | Direction | Maps to |
|---|---|---|---|
| 850 | Purchase Order | Inbound | PurchaseOrder + POLineItems |
| 810 | Invoice | Inbound | Vendor Invoice |
| 856 | Advance Ship Notice | Inbound | InventoryItem receipt |
| 855 | PO Acknowledgment | Outbound | PO status confirmation |
| 997 | Functional Acknowledgment | Outbound | Required for every inbound set |

**Healthcare transaction sets:**
| Set | Name | Direction | Maps to |
|---|---|---|---|
| 837P | Healthcare Claim (Prof) | Outbound | Invoice → claim with NPI, ICD-10, CPT |
| 835 | Remittance Advice | Inbound | Claim payments + adjustments → Invoice |
| 270 | Eligibility Inquiry | Outbound | Coverage check before service |
| 271 | Eligibility Response | Inbound | Coverage → Contract update |

**The 997 rule:** Every X12 inbound transaction set requires a 997 Functional Acknowledgment. The RULE `x12_always_send_997` fires on every validated envelope. No exceptions. Trading partners expect it. If you don't send it, they'll retransmit.

### UN/EDIFACT
The international counterpart to X12. European and global supply chains. Same concept, EDIFACT syntax:

```
ORDERS  = X12 850  (purchase order)
INVOIC  = X12 810  (invoice)
DESADV  = X12 856  (advance shipment notice)
REMADV  = X12 835  (remittance advice)
CONTRL  = X12 997  (functional acknowledgment)
```

EDIFACT uses `UNB/UNG/UNH` envelope segments vs X12's `ISA/GS/ST`. The validation logic is structurally identical. If you have European or global suppliers, you have both standards.

### RosettaNet PIPs
B2B interchange for high-tech manufacturing — semiconductor, electronics, aerospace. XML over HTTPS with digital signatures. Partners have certificates. Every message requires a signed signal response.

```
PIP 3A4  → Purchase Order (signed XML, requires ReceiptAcknowledgment)
PIP 3A7  → Change Purchase Order
PIP 3B2  → Advance Shipment Notification
PIP 3C3  → Invoice
```

The `SendRosettaNetSignal` rule fires on every validated PIP. Unlike X12's 997 (pure transport ACK), RosettaNet signals are cryptographically signed business-level acknowledgments.

---

## Eight Interchange Workflows

```
inbound_hl7_adt        → 7 steps: parse → validate → dedup → transform → apply → ACK → log
inbound_hl7_orm        → 7 steps: parse → validate → dedup → create_order → ACK → log
inbound_x12_850        → 8 steps: validate_envelope → validate_segments → dedup → transform → create_po → 997 → 855 → log
inbound_x12_810        → 7 steps: validate_envelope → validate_segments → match_po → transform → create_invoice → 997 → log
inbound_x12_856        → 6 steps: validate → validate → match_shipment → update_inventory → 997 → log
outbound_x12_837       → 5 steps: fetch_invoice → build_claim → validate → transmit → log
inbound_x12_835        → 7 steps: validate → parse → match_claims → post_payments → post_adjustments → 997 → log
inbound_edifact        → 7 steps: validate_envelope → parse → dedup → transform → apply → CONTRL → log
inbound_rosettanet_pip → 7 steps: validate_rnif → parse → dedup → transform → apply → signal → log
```

Every workflow is: validate the envelope, validate the segments, check for duplicates, transform to ERP entities, apply, acknowledge. The pattern is the same across all five standards. The spec differences are in the validation steps.

---

## The Deduplication Rule

```
RULE reject_duplicate_message {
  WHEN InterchangeContext.is_duplicate == true
  THEN FLAG "duplicate_message_rejected"
  SEVERITY warning
  PRIORITY 100
}
```

Interchange systems retransmit on timeout. You will receive the same 850 twice. The `CheckDuplicateMessage` action looks up by control number + partner ID + transaction type within a configurable window (default 24h). Duplicates are flagged and rejected before any ERP changes are made. The trading partner still gets their 997 ACK — so they stop retransmitting — but the ERP doesn't double-process.

This bug has cost companies millions of dollars in duplicate POs and payments. It's a named RULE now.

---

## Six Demo Trading Partners

```
Northgate Supplies        → vendor, X12/AS2        (supply chain)
Brightline Insurance      → payer, X12/SFTP        (healthcare claims/remittance)
Vertex EHR System         → EHR, HL7 v2.x/MLLP    (patient demographics, orders)
Acme FHIR Gateway         → EHR, FHIR R4/HTTPS     (modern clinical data)
Heidelberg GmbH           → vendor, EDIFACT/SFTP   (international supply chain)
Intel Supply Chain Portal → customer, RosettaNet   (high-tech manufacturing)
```

---

## Architecture

```
accelerando_interchange.agi
│
├── ENTITY × 4
│   TradingPartner       → partner config, connection details, ISA/MSH identifiers
│   InterchangeMessage   → every message with raw payload and processing status
│   MessageAcknowledgment → every ACK/997/CONTRL/signal sent
│   InterchangeError     → named errors with segment and field detail
│
├── STAGES InterchangeMessage.status
│   received → validated → transformed → applied → acknowledged / error
│
├── PACKET × 13          → typed schemas for every supported message type
│   HL7ADTPacket, HL7ORMPacket, HL7ORUPacket
│   FHIRResourcePacket
│   X12_850, X12_810, X12_856, X12_997, X12_837, X12_835, X12_270
│   EDIFACTPacket, RosettaNetPacket
│
├── CHANNEL × 17         → inbound and outbound per standard
│   hl7_adt_inbound / hl7_oru_outbound
│   fhir_inbound / fhir_outbound
│   x12_850_inbound / x12_810_inbound / x12_856_inbound
│   x12_810_outbound / x12_850_outbound / x12_997_outbound
│   x12_837_outbound / x12_835_inbound
│   edifact_inbound / edifact_outbound
│   rosettanet_inbound
│
├── MODULE × 6           → one per standard plus global engine
│   InterchangeEngine    → routing, dedup, partner validation
│   HL7Module            → 5 PATTERNs, 5 RULEs, HL7Flow STATE
│   FHIRModule           → 5 PATTERNs, 3 RULEs
│   X12Module            → 8 PATTERNs, 6 RULEs, EDIFlow STATE
│   EDIFACTModule        → 4 PATTERNs, 3 RULEs
│   RosettaNetModule     → 4 PATTERNs, 3 RULEs
│
├── WORKFLOW × 9         → full inbound/outbound processing sequences
│
├── ACTION × 28          → parse, validate, transform, apply, acknowledge per standard
│   GenerateInterchangeReport → AI: volume/error analysis + recommendations
│
├── VIEW × 5             → InterchangeDashboard, MessageQueue, ErrorQueue,
│                          TradingPartnerManager, AcknowledgmentLog
│
└── PREFERENCE × 3       → duplicate_detection_window, ack_timeout, max_message_size
```

---

## The Full Accelerando Stack

```
accelerando_erp.agi          → business data
accelerando_interchange.agi  → standard data interchange (this app)
accelerando_config.agi       → self-configuration
accelerando_chatbot.agi      → customer service
accelerando_eliza.agi        → operator interface
accelerando_es.agi           → governance
accelerando_oie.agi          → organizational intelligence
```

Seven `.agi` files. Every message that enters or leaves the system is typed, validated, transformed, acknowledged, and logged. The interchange is as auditable as the ERP. Every 850 that becomes a PurchaseOrder has a traceable path: raw EDI → named segments → mapped fields → ERP record → 997 ACK.

The ERP implementations you built in the 90s took months per interface. The spec was the same then as it is now. It just wasn't encoded yet.
