# Accelerando Compliance Training

**Compliance is not a training event. It is a knowledge state. This system measures the state. Daily.**

> Annual compliance training is not just ineffective.  
> In high-stakes domains, it is actively dangerous.

---

## The Problem With Annual Training

Ebbinghaus published the forgetting curve in 1885. Humans forget approximately 70% of new information within a week without reinforcement. This is not a controversial finding. It has been replicated continuously for 140 years.

Annual compliance training is designed around the assumption that this curve doesn't exist. A healthcare worker who passed HIPAA training on January 5th and has not been tested since is classified as "compliant" on December 4th. They have forgotten most of it. The checkbox says otherwise.

In low-stakes domains, this produces wasted training budgets and empty audit trails. In high-stakes domains — HIPAA, OSHA, safety protocols, insider trading prevention — it produces incidents. The training record says the employee was trained. The incident record says they didn't know what they were doing.

The regulatory intent behind compliance training is **knowledge retention**, not course completion. Annual training proves attendance. This system proves knowledge.

---

## The Better Model

Three questions. Sixty seconds. Every day.

```
Daily micro-assessment
  ↓
Question selected by spaced repetition algorithm
  ↓
Wrong answer → 5-minute targeted refresher surfaces immediately
  ↓
Score updates in real time → dashboard reflects current state
  ↓
Score drops below threshold → manager notified, refresher assigned
  ↓
Score recovers → compliance status restored
```

No 3-hour annual slog. No checkbox. No forgetting curve exploit.

The knowledge score is updated after every assessment using an exponential moving average:
```
new_score = (current_score × 0.85) + (result × 100 × 0.15)
```

A single wrong answer doesn't crater the score — the system is statistically robust against bad days. But consistent failure on a topic reliably drives the score below threshold, because it should. That's the point.

---

## Spaced Repetition — The Science Behind the Question Selection

`SelectDailyQuestions` doesn't pick random questions. It picks the questions each learner most needs to see today:

1. **Weakest domains first** — domains where `ComplianceScore.current_score` is lowest get prioritized
2. **Targeted subtopics** — when a learner misses a question, `weak_subtopic` is recorded; the next session serves questions from that subtopic
3. **Miss-weighted questions** — questions this learner has answered incorrectly appear more often than questions they've answered correctly
4. **Adaptive difficulty** — consecutive correct answers escalate difficulty; a struggling learner gets easier questions until they build confidence

The result: each learner's daily 3 questions are different from every other learner's, calibrated to their specific knowledge gaps. The employee who knows HIPAA cold but struggles with cybersecurity gets different questions than the one who's the reverse.

---

## Gamification — Why It Works

The biggest compliance training problem is not content quality. It's habit formation. Getting people to actually do the 60-second daily check-in, every day, without being nagged into it.

The same psychology that makes Duolingo work applies here:

**Streaks** — the loss aversion mechanic. People will protect a 47-day streak they wouldn't have started a 1-day streak. Two "streak freeze" credits are available per period — a sick day doesn't break everything.

**Points** — accumulate across all assessments, milestone badges at 500 / 2,000 / 5,000 / 10,000. Points don't decay. They're a record of consistent participation.

**Badges** — the shareable achievements. Streak milestones (7, 30, 90, 365 days), points milestones, Perfect Week (7 days all correct), Comeback (first completion after a broken streak — the system rewards returning, not just sustaining).

**Leaderboards** — by department. Public-facing compliance scores create positive peer pressure. The department that's at 94% average compliance is a different culture than the one at 71%.

**Streak Freeze Credits** — because life happens. Two per period. This is the mechanic that keeps the habit alive through unavoidable interruptions rather than letting one miss reset everything.

The gamification is not decoration. It is the mechanism that gets the 60-second daily check-in to happen. If it doesn't happen, the system doesn't work. If it does happen, the system produces provably compliant employees.

---

## Knowledge Scoring — What Compliance Actually Looks Like

The dashboard doesn't show "Alice completed HIPAA training on Jan 5." It shows:

```
Alice Chen — HIPAA Privacy
  Current Score:      87/100 ✓ Compliant
  Trend:              ↑ (+3 this week)
  Last Assessed:      Today
  Current Streak:     23 days
  Weak Subtopic:      PHI de-identification (assigned refresher)
  Refresher Status:   In progress (due in 4 days)
```

And for the department:

```
Nursing — ICU                    HIPAA Privacy    HIPAA Security    OSHA
  Compliant:          12 / 15       14 / 15          13 / 15
  At Risk:             2 / 15        1 / 15           1 / 15
  Non-Compliant:       1 / 15        0 / 15           1 / 15
  Avg Score:            82              88               79
```

When a manager opens this dashboard, they see the compliance state of their team **right now**. Not as of last January. Not as of whenever the annual training was. Now.

---

## The Refresher System — Targeted, Fast, Triggered

When a score drops below threshold, `AssignRefresher` fires. The system finds the `TrainingModule` that covers the specific `weak_subtopic` — not a 2-hour general review, a 5-minute targeted re-teaching of the exact concept the learner missed.

Refresher assignment → 7-day completion window (configurable). At day 7 with no completion:
- `RefresherContext.refresher_overdue = true`
- ES fires → PRIORITY 90 flag → manager notified with escalation

When a domain score drops below the critical threshold:
- Manager is notified immediately, not after the deadline
- The notification includes the score, the domain, the regulatory basis, and the refresher status
- This is not punitive. It is the manager's job to know when someone on their team is a compliance risk.

---

## Curriculum Generation — AI at Build Time

`GenerateQuestions` is an AI action. It runs once when onboarding a new compliance domain, and again when regulations change. It reads the training content and regulatory requirements and outputs Question records — structured, scored, explained.

The key constraint: the explanation field is the teaching moment. When a learner answers incorrectly, they don't just see "wrong." They see:

> **Why C is correct:** Under HIPAA's Minimum Necessary Rule, you should only request or disclose the minimum amount of PHI necessary to accomplish the intended purpose. Providing the full medical record when only the diagnosis is needed for treatment coordination exceeds the minimum necessary standard. This is one of the most common HIPAA violations in clinical settings.

That explanation, displayed immediately after an incorrect answer, is where the learning happens. Not in the 3-hour annual module. In the 15 seconds after getting something wrong, when the learner is curious about why.

`GenerateTrainingModule` creates the refresher content. 5 minutes, targeted to the subtopic, structured as: real-world stakes → core rule in plain language → concrete examples including a common mistake → key takeaway. Not a lecture. A teaching moment.

---

## Audit Export — What You Hand the Regulator

Annual training systems produce: a spreadsheet with employee names, training titles, and completion dates.

This system produces, per employee, per domain, for any point in time you specify:

- Current knowledge score
- Last assessment date
- All assessment responses for the period (question, answer, correct/incorrect, timestamp)
- Score trend over the period
- Refreshers assigned, started, completed
- Manager notifications sent
- Streak and participation record
- Immutable audit log with hash chain

The cover letter writes itself:
> *This system uses daily micro-assessment and continuous knowledge scoring rather than annual completion records. The attached data demonstrates that each employee was assessed on [regulatory domain] [N] times during the period, their knowledge score was maintained above [threshold] for [X]% of the period, and any scores below threshold triggered automatic remediation within [N] days.*

That is provably stronger evidence of compliance than a completion date. Regulators who understand adult learning theory know it. The ones who don't will.

---

## Ten Pre-Seeded Compliance Domains

| Domain | Regulatory Basis | Passing Score | Critical Score | Roles |
|---|---|---|---|---|
| HIPAA Privacy | 45 CFR Part 164 Subpart E | 80 | 65 | All |
| HIPAA Security | 45 CFR Part 164 Subpart C | 80 | 65 | All |
| OSHA General Safety | 29 CFR Part 1910 | 75 | 60 | All |
| Sexual Harassment Prevention | Title VII / state law | 80 | 65 | All |
| Cybersecurity Awareness | NIST SP 800-50 | 75 | 60 | All |
| Data Privacy (GDPR/CCPA) | GDPR Art 5 / CCPA 1798.100 | 75 | 60 | All |
| Anti-Bribery / FCPA | 15 U.S.C. § 78dd | 80 | 65 | All |
| SOX Financial Controls | SOX §302/404 | 75 | 60 | Finance, Exec, Accounting |
| Insider Trading Prevention | SEC Rule 10b-5 | 85 | 70 | Exec, Finance, Legal |
| Code of Business Conduct | Internal policy | 75 | 60 | All |

Run `GenerateQuestions` for each domain to populate the question bank. Run `GenerateTrainingModule` per domain per subtopic to build refresher content. The curriculum is generated once and updated when regulations change — not re-created manually every year.

---

## Architecture

```
accelerando_lms.agi
│
├── ENTITY × 10
│   ComplianceDomain        → domain registry: regulatory basis, thresholds, role applicability
│   TrainingModule          → content library: onboarding, refreshers, deep dives
│   Question                → question bank: scenario-based, with explanation for each answer
│   LearnerProfile          → learner record: streak, points, overall compliance, manager
│   ComplianceScore         → per-learner per-domain knowledge score (the real metric)
│   DailyAssessment         → daily assessment record: status, score, points, domains covered
│   AssessmentResponse      → per-question answer log: the immutable audit trail
│   RefresherAssignment     → triggered refresher: due date, completion status
│   ComplianceBadge         → earned achievements: streak milestones, points milestones
│   DepartmentCompliance    → rolled-up compliance rate per department per domain
│
├── STAGES
│   DailyAssessment.status  → pending → in_progress → completed / expired
│   RefresherAssignment     → assigned → in_progress → completed / overdue
│   TrainingModule.status   → draft → published → archived
│
├── MODULE × 6
│   AdaptiveAssessmentEngine → AssessmentContext FACT, AssessmentFlow STATE, deadline RULEs
│   ComplianceTracker       → ComplianceContext FACT, org_compliance SCORE, threshold RULEs
│   GamificationEngine      → GamificationContext FACT, learner_points SCORE, badge RULEs
│   RefresherEngine         → RefresherContext FACT, RefresherFlow STATE, escalation RULEs
│   CurriculumEngine        → CurriculumContext FACT, staleness and refresh RULEs
│   ReportingEngine         → ReportContext FACT, audit and regulatory report trigger RULEs
│
├── REASONER × 1
│   compliance_intelligence_reasoner → weekly: systemic gap detection, content effectiveness,
│                                       pattern analysis across departments and domains
│
├── WORKFLOW × 7
│   daily_assessment_cycle        → 8 steps: select → serve → score → update → streak → refresher → badges → log
│   onboard_new_employee          → 5 steps: profile → domains → initial assessment → notify → log
│   refresher_completion          → 4 steps: mark complete → update score → recalculate dept → log
│   daily_reminders               → 4 steps: identify pending → send reminders → expire missed → update streaks
│   weekly_compliance_rollup      → 5 steps: calc dept → check refreshers → summary → AI → log
│   generate_curriculum           → 4 steps: generate questions → generate modules → validate → publish
│   export_for_audit              → 3 steps: export records → generate report → log
│
├── ACTION × 13
│   GenerateQuestions              → AI (build-time): content → scenario-based question bank
│   GenerateTrainingModule         → AI (build-time): requirements → 5-minute targeted refresher
│   GenerateWeeklySummary          → AI: compliance data → CCO Monday morning briefing
│   GenerateRegulatoryComplianceReport → AI: period data → regulatory submission format
│   SelectDailyQuestions           → deterministic: spaced repetition → today's 3 questions
│   ScoreAssessment                → deterministic: responses → score + points + streak bonus
│   UpdateKnowledgeProfile         → deterministic: EMA score update + threshold checks
│   AssignRefresher                → deterministic: weak subtopic → targeted module assignment
│   NotifyManager                  → deterministic: critical score → manager notification
│   AwardBadge                     → deterministic: milestone detected → badge record
│   UpdateStreak                   → deterministic: completion status → streak + freeze logic
│   CalculateDepartmentCompliance  → deterministic: all scores → department rollup
│   ExportComplianceAuditRecord    → deterministic: point-in-time snapshot → audit package
│
├── VIEW × 8
│   ComplianceDashboard, LearnerView, DailyAssessmentView, ComplianceScoreView
│   RefresherQueue, QuestionBank, BadgeGallery, DomainManagement
│
└── PREFERENCE × 10
    daily_questions_per_learner (3), domain_passing_score (75), domain_critical_score (60),
    refresher_deadline_days (7), streak_freeze_credits_max (2),
    assessment_window_start/end (8–18), reminder_advance_hours (2),
    knowledge_decay_alpha (0.15), min_questions_per_domain (30)
```

---

## The Full Accelerando Stack

```
accelerando_erp.agi          → business data (32 entities, 32 actions)
accelerando_billing.agi      → medical billing (full claim lifecycle, self-updating rules)
accelerando_legal.agi        → eDiscovery and legal hygiene
accelerando_lms.agi          → compliance training (this app — daily micro-assessment, real-time scores)
accelerando_interchange.agi  → standard data interchange (HL7, FHIR, X12, EDIFACT, RosettaNet)
accelerando_config.agi       → self-configuration (13 templates, 6 advisory modules)
accelerando_chatbot.agi      → customer service (deterministic, cannot hallucinate)
accelerando_eliza.agi        → operator interface (20 workflows, macro executor)
accelerando_es.agi           → governance (34 rules, 6 policy modules)
accelerando_oie.agi          → organizational intelligence (AI reasoning, retrospective)
```

Ten `.agi` files. The LMS connects to the legal module (compliance scores can inform legal hold custodian risk assessment), to the ES (an employee with a critical HIPAA score can be flagged in the PHI access governance rules), and to the OIE (compliance training patterns are organizational intelligence — a department that's consistently failing cybersecurity questions is telling you something about their manager, their tooling, or their workload).

The compliance checkbox has been a liability for 140 years. It was only ever a proxy for the thing that actually mattered: does this person know what they're doing?

Now you can measure that. Daily.
