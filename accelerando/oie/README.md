# Accelerando OIE — Organizational Intelligence Engine

**"Crystal Reports for activity logs."**

Where Crystal Reports turned database records into formatted reports, the Accelerando OIE turns ERP and CRM activity logs into organizational intelligence. It runs as a standalone Tauri desktop app alongside an existing Accelerando instance — consuming telemetry from every service call, workflow, and chat interaction — and reasoning over it daily to surface what matters to leadership.

No custom code. No subsystem folders. No YAML schema files. One `.agi` file compiled by Agicore.

---

## The Problem It Solves

Accelerando captures everything: invoice creation, service desk tickets, quote approvals, workflow executions, chat commands. That activity data exists in the database — but nobody is watching it systematically. Patterns that would be obvious to a sharp operations analyst go unnoticed because surfacing them requires querying, aggregating, and interpreting data that nobody has time to query.

The OIE is the operations analyst that never sleeps.

It runs every night. It reads what happened. It reasons about what it means. It puts the findings in a dashboard that leadership can act on by morning.

---

## The Crystal Reports Analogy

| Crystal Reports | Accelerando OIE |
|---|---|
| Sits alongside an existing database | Sits alongside an existing ERP/CRM |
| Connects without modifying the source system | Consumes telemetry without modifying Accelerando |
| Turns raw rows into formatted reports | Turns raw activity into structured insights |
| Scheduled report generation | Daily + weekly reasoning jobs |
| Report templates define what to surface | SKILLDOC defines what "good insight" means |
| Role-based report access | AUTHORITY-governed insight access |
| Delivered to a specific audience | Scoped: GLOBAL / TEAM / USER |

The key difference: Crystal Reports presents data. The OIE interprets it.

---

## Architecture

```
Accelerando ERP/CRM
        │
        │  telemetry events (every service call, workflow, chat command)
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    TELEMETRY LAYER                               │
│  TelemetryEvent entity  ←  CHANNEL telemetry_ingress            │
│  SENSITIVE fields: input_summary, outcome_summary               │
│  PII never crosses the IPC bridge to the UI                     │
└─────────────────────────────────────────────────────────────────┘
        │
        │  4 reasoning cadences
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    REASONING LAYER                               │
│  daily_org_reasoner    — 24h batch, OIEAnalyst                  │
│  weekly_trend_reasoner — 7d cross-team, SeniorAnalyst           │
│  on_demand_summary     — leadership chat command                 │
│  personal_coach_reasoner — per-user coaching, PersonalCoach     │
│                                                                  │
│  NBVE shadow testing: sonnet vs opus                            │
│  ESCALATION_CHAIN: auto-escalate on SPC violation               │
└─────────────────────────────────────────────────────────────────┘
        │
        │  insights via CHANNEL insight_channel
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    GOVERNANCE LAYER                              │
│  QC_MESH InsightQuality                                         │
│  4 independent evaluators (claude-opus, gpt-4o, gemini-pro,     │
│    mistral-large) — majority consensus — > 5σ at full mesh      │
│  Insights that fail are flagged, not silently passed through     │
│                                                                  │
│  SKILLDOC oie_insight_standards                                  │
│  Non-engineers define what "good insight" means in prose.       │
│  The mesh evaluates against this document.                       │
└─────────────────────────────────────────────────────────────────┘
        │
        │  validated insights → OrgInsight entity
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LEADERSHIP CONSOLE                            │
│  OrgHealthDashboard  — top-line health view                     │
│  InsightReview       — split view: read + mark reviewed         │
│  BottleneckList      — table sorted by impact score             │
│  AutomationBoard     — kanban: pending → approved → dismissed   │
│  TelemetryMonitor    — raw event stream (admin only)            │
└─────────────────────────────────────────────────────────────────┘
```

---

## What Compiles from This Single File

```
accelerando_oie.agi
        │
        ▼  agicore compile
├── src-tauri/migrations/001_initial.sql
│     TelemetryEvent, OrgInsight, AutomationCandidate tables
│     Indexes on tenant_id, category, scope, event_type
│     SEED rows: 3 demo insights, 2 automation candidates
│
├── src-tauri/src/commands/
│     telemetry_event.rs    — read/list only (CRUD enforced)
│     org_insight.rs        — read/list/edit
│     automation_candidate.rs — read/list/edit + STAGES validator
│
├── src/lib/
│     cognition-roles.ts    — OIEAnalyst, SeniorAnalyst, PersonalCoach
│     escalation-chain.ts   — OIEChainEscalationChain engine
│     nbve.ts               — org_reasoner_nbve ShadowRunner + chain wire
│     qc-mesh.ts            — InsightQualityQcMesh + SPC sizing
│     stages.ts             — AutomationCandidate.status transitions
│
├── src/components/
│     OrgHealthDashboard.tsx
│     InsightReview.tsx
│     BottleneckList.tsx
│     AutomationBoard.tsx      — kanban, GROUP_BY status
│     TelemetryMonitor.tsx
│
├── scaffold/
│     cognition-org-chart.md
│     cognition-roles/OIEAnalyst.md
│     cognition-roles/SeniorAnalyst.md
│     cognition-roles/PersonalCoach.md
│     escalation-chains/OIEChain.md
│     qc-meshes/InsightQuality.md    — sigma table, integration guide
│     stages/AutomationCandidate_status.md
│
└── src-tauri/tauri.conf.json
      tray icon, global hotkey, capabilities ACL
```

---

## The Governance Stack

This is what separates the OIE from a dashboard that just shows data:

```
SKILLDOC oie_insight_standards
   ↓  defines what good insight looks like (prose, non-engineers own it)
QC_MESH InsightQuality
   ↓  4 evaluators, majority consensus, > 5σ combined miss rate
   ↓  SPC expands mesh from 2→4 evaluators when drift detected
OrgInsight entity
   ↓  only passes through if mesh approves (or flags for review)
Leadership Dashboard
```

At maximum mesh size (4 evaluators at 92% per-evaluator detection rate):
- Combined miss rate: 0.08⁴ ≈ 40 per million — exceeding 5σ
- SPC automatically reduces to 2 evaluators during stable periods (cost savings)
- Evaluator diversity is mandatory: different architectures, different training distributions

---

## Privacy by Construction

`input_summary` and `outcome_summary` on `TelemetryEvent` carry the `SENSITIVE` modifier. The Agicore compiler emits:

**Rust:** `#[serde(skip_serializing)]` — the field is stored in SQLite but never serialized across the IPC bridge.

**TypeScript:** `input_summary?: never` — the frontend type system makes it impossible to accidentally render the field.

The privacy boundary is structural, not a policy document. It cannot be accidentally bypassed.

---

## Demo Mode

The app ships with seed data — three insights and two automation candidates — so the dashboard is immediately useful without a live Accelerando connection. Open the app, see a populated leadership console, understand the value in 30 seconds.

To connect to a live Accelerando instance:
1. Set `oie.active_tenant` in Preferences to your tenant ID
2. Configure the `telemetry_ingress` channel endpoint to point to your Accelerando telemetry stream
3. Trigger `daily_org_reasoner` on-demand or wait for the overnight batch

---

## Extending the Intelligence

The OIE is designed to be extended without modifying application code:

**Add an insight category:** Edit `skilldocs/oie_insight_standards.md` — no code change required. The QC_MESH evaluates against the updated criteria automatically.

**Add a reasoning cadence:** Add a new `REASONER` block to the `.agi` file and recompile. The scheduler wires it automatically.

**Adjust governance:** Update `AUTHORITY` levels or `QC_MESH` consensus threshold in the `.agi` file. Recompile.

**Promote a cheaper model:** The `NBVE` shadow runner handles this automatically via SPC. When `sonnet` consistently matches `opus` quality across 50 runs, it promotes without human intervention.

---

## Relationship to the organizational-intelligence Example

`examples/organizational-intelligence/` is a compact pattern demo — a single-file illustration of the OIE concept using a subset of Agicore primitives.

This application is the full implementation:

| | `examples/` | `apps/accelerando-oie/` |
|---|---|---|
| Entities | 1 (InsightRecord) | 3 (TelemetryEvent, OrgInsight, AutomationCandidate) |
| Reasoners | 2 | 4 |
| Cognition Roles | — | 3 (OIEAnalyst, SeniorAnalyst, PersonalCoach) |
| Escalation | — | ESCALATION_CHAIN + NBVE CHAIN |
| Insight governance | — | QC_MESH (4 evaluators, SPC-governed) |
| Privacy | — | SENSITIVE fields |
| Workflow | — | STAGES (automation candidate review) |
| Demo data | — | SEED (3 insights, 2 candidates) |
| Views | 2 | 5 |

---

## Related Reading

- [`NULLCLAW.md`](../../NULLCLAW.md) — agent runtime that can emit telemetry as a first-class operation
- [`CHANNEL.md`](../../CHANNEL.md) — typed message queues for telemetry ingestion
- [`REASONING.md`](../../REASONER.md) — REASONER declaration reference
- [`SKILLDOCS.md`](../../SKILLDOCS.md) — governance for cognition modules
- [`COGNITION_ROLE.md`](../../COGNITION_ROLE.md) — organizational cognition hierarchy
