# NovaSyn MBA

A portfolio management OS for solo founders, built on the [Agicore](https://github.com/chrisbender/agicore) 4G dev stack. Manage every business you run — from raw idea to exit — with seven AI discipline advisors, deterministic ERP-style tracking, and a cross-portfolio synergy engine.

Inspired by *The One-Person Enterprise: The Complete MBA for the AI-Powered Solo Founder* — the free-tier course and textbook from the **[AI WIN-WIN Institute](https://www.skool.com/tao-team-ai-outreach-4986)**, Christopher Bender's AI business school on Skool. The community there is the home base for ongoing NovaSyn MBA development, feedback, and discussion.

---

## What It Is

Most solo founders run multiple bets at once — a consulting practice, a SaaS product, a course, a community. They track each one in a different spreadsheet, a different Notion page, a different brain region. Nothing talks to anything else.

NovaSyn MBA gives you a single desktop OS for the whole portfolio:

- **Business lifecycle tracking** — every business moves through a defined stage gate (Idea → Validated → Building → Active → Scaling → Exited), with AI-enforced transition criteria
- **Seven AI discipline advisors** — one per MBA function (Strategy, Operations, Finance, Marketing, Sales, Technology, Leadership), each trained on the frameworks from *The One-Person Enterprise*
- **$20/$200 Audit tool** — classify every task by execution tier, identify what to delegate or automate, track transfer status
- **Unit economics dashboard** — CAC, LTV, LTV:CAC ratio, gross margin, MRR, and cash runway per business and across the portfolio
- **Content-to-client pipeline tracker** — kanban for the full authority marketing funnel from idea to published asset to inbound lead
- **Sales pipeline** — contact tracking from prospect to closed, with referral architecture built in
- **Cross-portfolio synergy detection** — AI scans all businesses simultaneously and surfaces shared audiences, tech leverage, distribution opportunities, and content reuse that no single-business view can see

---

## Built to Stress-Test Agicore

NovaSyn MBA is also a reference application for the Agicore framework. It is deliberately scoped to exercise every major feature of the compiler pipeline:

- Complex multi-entity schemas with BELONGS_TO / HAS_MANY relationships
- COMPILER-driven state machines (the business lifecycle)
- AI_SERVICE actions that cross entity boundaries (portfolio synergy detection)
- SEED data for immediate demonstrability on first launch
- WORKFLOW orchestration for business onboarding sequences
- Pending framework features are marked as `## TODO(agicore)` comments in the `.agi` file, making this app a living feature request tracker for the framework

See `DESIGN.md` for the full architecture, entity model, and Agicore framework stress-test goals.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | [Agicore](https://github.com/chrisbender/agicore) (Tauri + Rust + React + SQLite) |
| DSL | `.agi` source file compiled to migrations, Rust commands, TS types, Zustand store, React components |
| AI | Claude (Anthropic API) with optional OpenAI fallback |
| Desktop | Tauri v2 (Windows / macOS / Linux) |
| UI | React + Tailwind, dark theme |

---

## Getting Started

**Prerequisites**: Rust toolchain, Node.js 20+, Tauri CLI, Agicore compiler

```bash
# 1. Clone the Agicore monorepo
git clone https://github.com/chrisbender/agicore
cd agicore

# 2. Set your Anthropic API key
export ANTHROPIC_API_KEY=sk-ant-...

# 3. Compile the DSL
cargo run --bin agicore-compiler -- apps/novasyn-mba/novasyn_mba.agi

# 4. Run the app in development mode
cd apps/novasyn-mba
npm install
npm run tauri dev
```

On first launch, SEED data loads a sample business ("Clarity Consulting") so the app is immediately explorable without manual data entry.

---

## Community

The **[AI WIN-WIN Institute](https://www.skool.com/tao-team-ai-outreach-4986)** on Skool is the HQ for NovaSyn MBA. The AI MBA course and *The One-Person Enterprise* textbook that this app is built from are available on the free tier. Join the community to follow development, suggest features, and connect with other solo founders using the app.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Bug reports, feature requests, and Agicore framework gap reports are all welcome.

---

## License

MIT — see [LICENSE](LICENSE).

Copyright (c) 2026 Christopher Bender / NovaSyn.
