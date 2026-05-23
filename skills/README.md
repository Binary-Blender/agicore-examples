# Accelerando — Consulting Skill Docs

This directory ships **packaged consulting expertise** for advising real-world companies on deploying the Accelerando enterprise stack. Each skill doc is a single self-contained Markdown file you can attach to any AI assistant (Claude, GPT, an open-source model running locally) to turn it into a competent practitioner in this domain.

The docs follow the [Skill Doc format spec](https://github.com/Binary-Blender/agicore/blob/main/skills/SKILL_FORMAT.md) (v1.1+) maintained in the Agicore framework repo.

---

## What's here

### [`accelerando.manufacturing.baby.skill.md`](accelerando.manufacturing.baby.skill.md) — ~8.3k tokens

**Audience:** small open-source models (7B–13B class), tight context windows, fast iteration loops.

**Domain:** advising mid-sized discrete manufacturers (100–300 employees, $30–80M revenue) on deploying or replacing their ERP using the Accelerando 12-app Enterprise Core stack.

Catalog of all 12 Enterprise apps in manufacturing context. The "Acme Machining" 18-month deployment walked end-to-end. 10 anti-patterns from real ERP failures (Hershey 1999, FoxMeyer 1996, HP 2004, Lidl 2018) with mitigations. 5 rubric self-checks.

### [`accelerando.manufacturing.super.skill.md`](accelerando.manufacturing.super.skill.md) — ~25k tokens

**Audience:** frontier models (Claude Opus/Sonnet, GPT-4/5 class, Gemini Pro), long-context authoring sessions, deep design work.

**Domain:** same as Baby, broader range (50–500 employees, $10–250M revenue).

5 industry archetypes with end-to-end playbooks: greenfield discrete, legacy ERP replacement, multi-plant rollout, M&A plant integration, customer-pressure rescue. Per-app deep deployment guidance with configuration decisions, integration patterns, common configurations per archetype, gotchas. KPI framework with target ranges per role. Change-management playbook. 20 anti-patterns. Edge cases (process manufacturing differences, regulated industries explicitly out of scope, multi-currency, Tier-1 vs Tier-2). 10 rubric self-checks.

---

## How to use a skill doc

1. **Attach it to your AI session** as a system prompt or a file:
   - In Claude Code: `claude code --system "$(cat accelerando.manufacturing.baby.skill.md)" ...`
   - In Claude.ai: drag the file into the conversation as an attachment.
   - In Cursor: drop it in `.cursorrules` or reference it from your project's rules.
   - In an open-source model running locally: prepend it to your system prompt.

2. **Ask domain questions:**
   - "I run a 200-person CNC shop on QuickBooks. Should I deploy Accelerando? Walk through the 18-month plan."
   - "Acme Industries just acquired a 75-person specialty grinder shop. How do we integrate?"
   - "Critique this deployment plan: [paste]. Which anti-patterns does it hit?"
   - "My CFO asks how he'll know the $1.5M deployment is working. What KPIs do I show him?"

3. **The model verifies its own output against L6 rubrics.** Each self-check prompt ships with an explicit numbered checklist of items the response must address. The model can self-grade.

---

## Why this exists

Consulting expertise for ERP deployments normally lives in the heads of $400/hr Big-4 implementation partners. Packaging it as a portable Markdown file means a small manufacturer who couldn't afford McKinsey or Deloitte gets the same playbook quality for the cost of a Claude subscription.

Every IATF 16949–eligible Tier-2 shop in America now has access to deployment guidance grounded in real ERP failure patterns (Hershey, FoxMeyer, HP, Lidl) with named mitigations. The framework is open source; the consulting is open source; the cost gate just dropped to near zero.

---

## What's NOT in this skill doc family yet (and why)

- **Process manufacturing** (chemicals, food, ingredients) — different ontology (batches, recipes, lots, FIFO, expiration). Needs its own skill doc.
- **Healthcare / EMR** — the Accelerando EMR stack (6 apps in `agicore-examples/accelerando/`) requires HIPAA/HITRUST attestation work outside what these docs scope. A future regulated-industries skill doc will cover.
- **Defense / ITAR / Nuclear** — specialized infrastructure requirements; out of v1.0 scope.
- **Services businesses** — different stack composition; service-business skill doc planned.

If you want to contribute one of the missing skill docs, open an issue in the [main agicore repo](https://github.com/Binary-Blender/agicore/issues) — the format spec is portable and the pattern is established.

---

## Related

- **The Accelerando apps** themselves: see [`../accelerando/`](../accelerando/) in this repository. Each app is a single `.agi` file that compiles to a running Tauri/Axum application.
- **The Agicore DSL skill docs** (for authoring `.agi` files): see [`agicore.baby.skill.md`](https://github.com/Binary-Blender/agicore/blob/main/skills/agicore.baby.skill.md) and [`agicore.super.skill.md`](https://github.com/Binary-Blender/agicore/blob/main/skills/agicore.super.skill.md) in the main agicore repo.
- **The format spec**: [`SKILL_FORMAT.md`](https://github.com/Binary-Blender/agicore/blob/main/skills/SKILL_FORMAT.md) — v1.1 introduced the **rubric** self-check mode that these consulting docs use (the Agicore DSL docs use **mechanical** self-check, verified by the parser).
