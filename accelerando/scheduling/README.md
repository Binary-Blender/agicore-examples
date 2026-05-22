# Accelerando Scheduling

**No double bookings. No forgotten recalls. No show-up and wait.**

> The most common scheduling failure is not the double booking the system catches.  
> It's the recall that went unsent because nobody flagged it as overdue.  
> It's the high-priority patient who sat on the waitlist while a low-priority patient took the slot.  
> It's the provider who hit 30 patients at 2pm and never recovered.

---

## What This Solves

Scheduling systems book appointments. This one manages the entire patient access lifecycle:

- **Template enforcement** — every appointment validates against the provider's schedule template before write. No appointments outside template hours. No appointments without buffer time.
- **Double booking prevention** — PRIORITY 100. Slot validation happens before the record is written, not after.
- **Waitlist intelligence** — when a cancellation opens a slot, the waitlist is sorted by priority and candidates are notified automatically. The slot is held for 60 minutes.
- **Recall management** — recall tasks created at encounter close are tracked to outreach completion. Three attempts without response escalates to provider review. High-priority recalls (elevated clinical urgency) fire at PRIORITY 92.
- **No-show tracking** — patient reliability scores decay with each no-show. Chronic no-show patients trigger provider review. No-shows without reminders fire a process gap flag — not a patient compliance flag.

---

## The Double Booking Rule

```
RULE double_booking_prevention {
  WHEN SchedulingContext.double_booking_detected == true
  THEN FLAG "double_booking_blocked_slot_already_occupied"
  SEVERITY critical
  PRIORITY 100
}
```

`ValidateSlot` runs before `CreateAppointmentRecord`. If the slot is occupied — whether by a confirmed appointment, a waitlist hold, or a schedule exception — the booking is blocked. The validation is separate from the write so the check cannot be skipped.

---

## Recall Outreach — The Scheduled Follow-Through

The recall pipeline:

```
Encounter closes → RecallTask created with due_date
       ↓
Due date reached → recall_outreach workflow fires
       ↓
GenerateRecallMessages (AI: personalized per patient, reason, and attempt number)
       ↓
SendOutreach → RecordOutreachAttempt
       ↓
3 attempts, no response → FLAG "three_recall_attempts_no_response_provider_review"
       ↓
Provider reviews → lost_to_followup or new outreach method
```

`GenerateRecallMessages` is AI at build time — the message is generated fresh per patient, not a mail-merge template. The system tracks outreach method, attempt number, and response — so the third-attempt message reads differently from the first.

---

## No-Show Management

```
RULE repeated_no_show {
  WHEN NoShowContext.repeated_no_show_patient == true
  THEN FLAG "repeated_no_show_patient_escalate_to_provider"
  SEVERITY warning
  PRIORITY 80
}

RULE no_show_without_reminder {
  WHEN NoShowContext.no_show_without_reminder == true
  THEN FLAG "no_show_occurred_without_reminder_sent_process_gap"
  SEVERITY warning
  PRIORITY 72
}
```

The second rule is the important one. When a patient no-shows without receiving a reminder, the flag is on the process — not the patient. The patient reliability score does not decrease for a no-show where the reminder wasn't sent.

---

## Architecture

```
accelerando_scheduling.agi
│
├── ENTITY × 8
│   Provider, ScheduleTemplate, AppointmentType, Appointment
│   WaitlistEntry, RecallTask, Room, ScheduleException, NoShowRecord
│
├── STAGES
│   Appointment → requested → confirmed → checked_in → in_progress → completed / cancelled / no_show
│   WaitlistEntry → waiting → offered → accepted → expired
│   RecallTask → pending → outreach_sent → appointment_scheduled → completed / lost_to_followup
│
├── PACKET × 4
│   AppointmentConfirmedPacket → clinical, billing, patient portal
│   AppointmentCancelledPacket → clinical, billing, patient portal
│   RecallDuePacket → patient portal
│   NoShowAlertPacket → clinical, population health
│
├── MODULE × 4
│   SchedulingEngine  → double booking, resource conflict, capacity, buffer rules
│   RecallEngine      → overdue detection, three-attempt escalation, high-priority rule
│   WaitlistEngine    → cancellation slot matching, priority ordering, hold management
│   NoShowEngine      → patient reliability score, chronic detection, process gap flag
│
├── WORKFLOW × 6
│   book_appointment, cancel_appointment, reschedule_appointment
│   recall_outreach, process_no_show, waitlist_fill
│
└── ACTION × 20
    FindAvailableSlots, ValidateSlot, CheckResourceAvailability  → deterministic
    CreateAppointmentRecord, ScheduleReminders, EmitConfirmationPacket → deterministic
    NotifyWaitlist, FindWaitlistCandidates, HoldSlotForWaitlist → deterministic
    FindOverdueRecalls, SendOutreach, RecordOutreachAttempt → deterministic
    MarkAppointmentNoShow, UpdatePatientReliabilityScore → deterministic
    GenerateRecallMessages → AI (claude-haiku): personalized per attempt
    OptimizeScheduleTemplate → AI (claude-sonnet): utilization analysis
    AssessScheduleHealth → deterministic: utilization metrics
```

---

## Downstream Integration

Scheduling feeds everything else:

- `AppointmentConfirmedPacket` → **Clinical** (pre-populate encounter), **Billing** (expected revenue), **Patient Portal** (appointment view)
- `AppointmentCancelledPacket` → **Billing** (remove expected revenue), **Patient Portal** (update calendar)
- `NoShowAlertPacket` → **Clinical** (flag for provider), **Population Health** (no-show pattern for risk scoring)
- **Scheduling** receives `ReferralPacket` from Clinical to route specialty appointments

---

## Seed Data

8 pre-seeded appointment types:

| ID | Type | Duration | Telehealth |
|---|---|---|---|
| apt-001 | New Patient | 60 min | Yes |
| apt-002 | Follow-Up | 20 min | Yes |
| apt-003 | Annual Wellness Visit | 45 min | No |
| apt-004 | Urgent Visit | 30 min | Yes |
| apt-005 | Procedure | 60 min | No |
| apt-006 | Telehealth Visit | 20 min | Yes |
| apt-007 | Pre-Op Clearance | 45 min | No |
| apt-008 | Post-Op Follow-Up | 30 min | Yes |
