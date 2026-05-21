# Accelerando Process Improvement CoE

**TPS. Six Sigma. Kaizen. And a system that will not let you slide back six months later.**

> The improvement part is not the hard part.  
> Getting people in a room for 4 days, mapping the value stream, identifying waste,  
> building an action list — that's the easy part. People are good at this.  
>  
> The hard part is six months later.

---

## The Backslide Problem

It has a name in manufacturing. "Kaizen Theatre." The event happens, the before-and-after photos go on the wall, the metrics improve, and then — gradually, invisibly — the process drifts back. The standardized work gets modified by one operator, then another. The kanban cards accumulate in a drawer. The control chart nobody updates. The champion who got promoted and took the improvement with them.

The 30-day check shows the improvement held. Then everyone moves on. Nobody schedules the 60-day check.

This system schedules the 60-day check. And the 90-day check. And the 6-month check. And the annual check. And it measures the metrics continuously. And it notices when drift begins before it becomes regression. And it nags. Systematically. Indefinitely. Because the purpose of a Kaizen is not to have a Kaizen — it is to have a permanently better process.

---

## The Anti-Backslide Core

`improvement_sustainability` is the central score. It decays.

```
SCORE improvement_sustainability {
  INITIAL 100
  MIN       0
  MAX     100
  DECAY     1 PER week          // sustained improvements require active effort
  THRESHOLD healthy   AT 85 THEN FLAG "improvement_well_sustained"
  THRESHOLD drifting  AT 70 THEN FLAG "improvement_drifting_intervention_required"
  THRESHOLD regressed AT 50 THEN FLAG "improvement_regressed_kaizen_review_required"
}
```

1 point per week decay. With no sustaining activity, you hit "drifting" at 15 weeks — which is almost exactly when real-world backsliding becomes visible. The score recovers when sustainability checks are completed and metrics are confirmed. Active sustaining keeps the score healthy. Neglect lets it drift to exactly the state you're trying to prevent.

The five failure modes are each tracked separately as `SustainabilityContext` boolean flags:

| Flag | What It Catches |
|---|---|
| `check_overdue` | Nobody scheduled the 30/60/90 day follow-up |
| `metric_regression_detected` | The metric is drifting back toward baseline |
| `control_audit_overdue` | The control plan isn't being audited |
| `champion_reassigned` | The person who drove it left — ownership gap |
| `standardized_work_drifted` | People stopped following the SWI |

Each flag fires a different RULE at a calibrated priority. The champion continuity flag fires at PRIORITY 88 — below a metric regression (PRIORITY 95) but above a missed check (PRIORITY 85) — because it's the root cause of most of the others.

---

## Sustainability Checks — The Scheduled Nag Sequence

When a Kaizen event closes, `CreateSustainabilityChecks` automatically schedules:

```
30 days  → metric confirmed, action items complete, control plan in place
60 days  → metric trend analysis, control audit completed
90 days  → final sustainability gate — sustained or regressed
6 months → ongoing hold (quarterly detection)
Annual   → permanent monitoring
```

Each check is assigned to the Kaizen champion. If the check isn't completed 7 days before it's due, a reminder fires. If it isn't completed by the due date, it escalates. The escalation goes to the champion **and** the area sponsor. Because if the champion forgot, the sponsor needs to know.

If the 90-day check passes, `EmitReplicationPacket` fires — the improvement is documented for replication to other value streams.

If the 90-day check fails, `KaizenEvent.status = "regressed"` and a sustainability review is triggered. Not a new Kaizen. A review of what broke down in the control plan that allowed the regression.

---

## Metric Drift Detection

`DetectMetricDrift` compares the rolling 7-day average to both the post-event target and the pre-event baseline, and applies Nelson Rules for SPC signals:

```
regression_pct = (current_avg - target_value) / (baseline_value - target_value)

< 15% regression → "minor" → review control plan
15–50% regression → "major" → sustainability review required  
≥ 50% regression → "critical" → emit KaizenRegressionAlert immediately
```

A 50% regression means you're halfway back to where you started. The system doesn't wait for 100%.

Nelson Rules also run on every MetricReading:
- Rule 1: Point outside 3σ
- Rule 2: 9 consecutive points same side of centerline
- Rule 3: 6 consecutive points trending in one direction

Rule 3 is the early warning. A trend toward regression is detectable 3–4 weeks before it crosses a threshold. The system flags the trend, not just the crossing.

---

## 5S — The Score That Decays Faster

5S decays faster than other improvements because the Sustain (the 5th S) is the one nobody does.

```
SCORE five_s_score {
  INITIAL   0
  MIN       0
  MAX     100
  DECAY     3 PER week    // 5S degrades visibly without active sustaining
  THRESHOLD world_class AT 90 THEN FLAG "five_s_world_class_maintain"
  THRESHOLD acceptable  AT 75 THEN FLAG "five_s_zone_acceptable"
  THRESHOLD declining   AT 60 THEN FLAG "five_s_declining_schedule_audit"
  THRESHOLD critical    AT 40 THEN FLAG "five_s_critical_5s_event_required"
}
```

3 points per week. From world class (90), you'd hit "declining" in 10 weeks without a 5S audit. That's realistic. A clean, organized shop becomes a disaster in 2–3 months without the sustaining habits — and the sustaining habits don't form without the audit.

The `FiveSSurvey` entity scores each S separately (0–20 each, max 100). The sustain score is always the hardest to earn, because it requires evidence of ongoing habits, not just a clean floor on audit day.

---

## TPS Principles as Expert System Rules

The TPS knowledge isn't stored in a document library. It's encoded as governance rules that fire when violations occur.

**Jidoka — Stop the Line:**
```
RULE jidoka_stop_line {
  WHEN TPSContext.andon_triggered == true
  AND  TPSContext.defect_rate_above_threshold == true
  THEN FLAG "stop_line_quality_threshold_exceeded_jidoka"
  SEVERITY critical
  PRIORITY 100
}
```

**Kanban — Overproduction Detection:**
```
RULE overproduction_kanban_violated {
  WHEN TPSContext.wip_above_kanban_limit == true
  THEN FLAG "overproduction_kanban_limit_exceeded_pull_signal_required"
  SEVERITY warning
  PRIORITY 85
}
```

**Takt Time — Flow Monitoring:**
```
RULE takt_time_violation {
  WHEN TPSContext.cycle_time_above_takt == true
  THEN FLAG "cycle_time_exceeds_takt_flow_at_risk_kaizen_candidate"
  SEVERITY warning
  PRIORITY 83
}
```

These aren't alerts you configure. They're the Toyota Production System, encoded. The system knows what overproduction looks like. It knows what a Jidoka trigger looks like. It knows when flow is at risk.

---

## Six Sigma DMAIC — Phase Gate Enforcement

DMAIC projects can't advance without completing the prior phase. The tollgates are enforced.

| Phase Gate | What's Required |
|---|---|
| Define → Measure | Charter signed, sponsor engaged |
| Measure → Analyze | Baseline capability study complete |
| Analyze → Improve | Root cause confirmed |
| Improve → Control | Solution statistically validated |
| Control → Closed | Control plan complete, standard work updated, sustainability checks scheduled |

```
RULE control_phase_gate {
  WHEN SigmaContext.control_plan_complete == false
  AND  SigmaContext.solution_validated == true
  THEN FLAG "control_phase_control_plan_required_before_closure"
  SEVERITY critical
  PRIORITY 95
}
```

A project cannot close without a control plan. This is the most commonly skipped step in Six Sigma implementations. The team declares victory, dissolves, and the improvement regresses because nobody built the sustaining system. This system won't let that happen.

---

## The 8 Wastes — Pre-Seeded for Waste Walks

All 8 wastes of Lean are seeded as `WasteObservation` templates:

| Waste | Common Manifestation |
|---|---|
| Transportation | Material moving between non-adjacent workstations |
| Inventory | WIP accumulation exceeding kanban limit |
| Motion | Operator reaches above shoulder or below knee for frequent items |
| Waiting | Machine waits for operator — operator waits for machine |
| Overproduction | Producing to forecast instead of to customer pull signal |
| Over-processing | Tighter tolerance than customer specification requires |
| Defects | Rework station exists — rework is designed into the process |
| Skills | Operators performing non-value-add tasks their skills exceed |

The rework station observation is the sharpest one. If a rework station exists in your process, rework is designed into the process. The process assumes defects. That is the waste — not the rework station itself.

---

## AI at Build Time, Determinism at Runtime

**`GenerateKaizenCharter`** — AI reads the problem statement and generates a structured 4-day event charter: problem statement, SMART goals, scope (including explicit out-of-scope — scope creep kills Kaizens), team composition, pre-work, agenda, and success criteria.

**`GenerateControlPlan`** — AI generates the control plan from the improvement description: critical parameters, control methods (SPC chart type, measurement frequency, responsible party), reaction plan (specific steps — "investigate" is not a reaction plan), audit schedule, and visual controls.

**`GenerateStandardWork`** — AI generates the SWI: Standard Work Combination Sheet, sequence of operations, cycle times, quality checks, safety points, key points, and — most importantly — the *reasons* for the key points. Operators who understand why follow the standard. Operators who don't skip the "unnecessary" steps.

**`AnalyzeDriftPattern`** — AI reads the metric time series and classifies the regression: random variation vs. systematic drift, severity, root cause hypothesis (time-based habit drift vs. step-change event vs. seasonal pattern), and recommended intervention.

**`SuggestImprovementOpportunities`** — AI ranks improvement opportunities by impact × feasibility from waste observations and metric data. The highest-ranked opportunity is not always the biggest — a visible quick win builds the culture that makes the harder improvements possible.

---

## Architecture

```
accelerando_pi_coe.agi
│
├── ENTITY × 14
│   ValueStream, ImprovementProject, KaizenEvent, ActionItem
│   ProcessMetric, MetricReading, ControlPlan, ControlAudit
│   StandardWork, WasteObservation, SustainabilityCheck
│   BeltCertification, LessonsLearned, FiveSSurvey
│
├── STAGES
│   ImprovementProject → define → measure → analyze → improve → control → closed
│   KaizenEvent        → planning → active → follow_up → sustained / regressed
│   ActionItem         → open → in_progress → complete / overdue
│   SustainabilityCheck → scheduled → complete / missed / escalated
│
├── PACKET × 3
│   ReplicationPacket         → sustained improvements to other value streams
│   KaizenRegressionAlert     → critical regression to CoE leadership
│   NCRTriggerPacket          → defect patterns to QMS for nonconformance
│
├── CHANNEL × 3
│   replication_outbound, regression_alert_outbound, ncr_trigger_outbound
│
├── MODULE × 7
│   KaizenEngine          → KaizenContext FACT, KaizenFlow STATE, action item RULEs
│   SustainabilityEngine  → SustainabilityContext FACT, improvement_sustainability SCORE,
│                           5 backslide detection RULEs
│   TPSEngine             → TPSContext FACT, five_s_score SCORE, Jidoka/Kanban/Takt RULEs
│   SixSigmaEngine        → SigmaContext FACT, process_sigma SCORE, DMAIC tollgate RULEs
│   MetricsEngine         → MetricsContext FACT, oee_score SCORE, SPC RULEs
│   ReplicationEngine     → ReplicationContext FACT, lessons learned and replication RULEs
│   COEManagement         → COEContext FACT, portfolio_health_score SCORE
│
├── REASONER × 1
│   process_intelligence_reasoner → weekly: systemic patterns, replication opportunities,
│                                   regression risks, waste walk recommendations, belt utilization
│
├── WORKFLOW × 8
│   run_kaizen_event, dmaic_define, dmaic_measure, dmaic_control
│   sustainability_check, five_s_audit, waste_walk_and_prioritize, capture_and_replicate
│
├── ACTION × 12
│   GenerateKaizenCharter       → AI: problem statement → 4-day event charter
│   GenerateControlPlan         → AI: improvement → control plan with reaction steps
│   GenerateStandardWork        → AI: process description → SWI with key point reasons
│   AnalyzeDriftPattern         → AI: metric series → regression classification + root cause
│   SuggestImprovementOpportunities → AI: waste observations → ranked project list
│   GenerateLessonsLearned      → AI: project data → replicable lessons
│   CalculateProjectROI         → deterministic: metric delta → $ value by type
│   CreateSustainabilityChecks  → deterministic: Kaizen close → 30/60/90/6m/annual schedule
│   DetectMetricDrift           → deterministic: readings → Nelson Rules + regression %
│   RunSPCAnalysis              → deterministic: readings → Cpk + out-of-control signals
│   ScoreFiveSSurvey            → deterministic: survey responses → 5S score update
│   EmitReplicationPacket       → deterministic: sustained + ROI → cross-VS replication
│
└── PREFERENCE × 12
    sustainability check intervals, regression thresholds, replication ROI minimum,
    control audit frequency, 5S target, OEE world-class threshold
```

---

## Connected to the QMS

The PI CoE and QMS are the two-stroke engine of manufacturing quality:

- **NCRs with recurring root causes** → `ReferToPICoE` → Kaizen event triggered
- **QMS internal audit findings** → systemic issues escalated to PI CoE as improvement projects
- **Kaizen results** → `EmitNCRTriggerPacket` → QMS captures process-level nonconformances as evidence of the improvement baseline
- **Completed Kaizens** → control plans register with QMS document control
- **OIE** → REASONER analyzes quality and improvement data together as organizational intelligence

```
accelerando_pi_coe.agi  ←→  accelerando_qms.agi
```

The QMS tells you what's wrong. The PI CoE fixes it permanently.
