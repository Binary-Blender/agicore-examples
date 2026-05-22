# Example 01 — Research Pipeline

A research assistant that takes a topic, runs parallel web searches, summarizes each source individually, then synthesizes a final structured report. Compiles to a Tauri desktop application.

---

## What problem this solves

Knowledge workers spend hours reading sources, taking notes, and writing summaries before they can synthesize anything useful. This example automates that loop: give it a research topic and query, and it fans out to multiple sources in parallel, summarizes each one with a focused AI call, then synthesizes the summaries into a structured report with executive summary, key findings, gaps, and next steps.

---

## The agentic anti-pattern it replaces

The LangChain version of this is a ReAct agent with a web search tool, a read-URL tool, and a summarize tool chained together in a prompt loop. Common failure modes:

- The agent "decides" to summarize only 2 of 5 sources because token pressure pushes it to shortcut
- A single bad URL crashes the entire chain with no recovery
- The synthesis step receives a mix of raw HTML, markdown, and half-completed summaries with no type boundary
- There is no record of which sources were used, what each summary said, or which model ran which step

Here, every source is a typed `Source` entity in SQLite. Every summary is stored. The synthesis step receives a typed JSON payload. If summarization fails for one source, `ON_FAIL skip` continues without crashing the workflow. Everything is auditable by querying the database.

---

## Key Agicore declarations

**`ENTITY ResearchTopic`, `Source`, `Report`**
The data model is the audit trail. Every research run creates rows in SQLite. You can inspect every source URL and summary after the fact. The `BELONGS_TO` relationships enforce referential integrity at the schema level.

**`AI_SERVICE` with two models**
`claude-sonnet-4-20250514` handles synthesis (high-quality, streaming). `gpt-4o-mini` is available as a faster/cheaper option for bulk summarization. The model selection is a declaration — changing it does not require touching application code.

**`ACTION search_web`, `summarize_source`, `synthesize_report`**
Each action has a typed `INPUT` and `OUTPUT`. The compiler generates Tauri commands with Rust type signatures. If `search_web` returns malformed JSON, the type boundary catches it before `synthesize_report` receives it.

**`WORKFLOW research_workflow`**
Three steps in declaration order: gather sources, run parallel summaries, build report. `ON_FAIL stop` on the first and last steps — a bad topic or a synthesis failure are real errors. `ON_FAIL skip` on the summarization step — a single bad URL should not abort the entire pipeline.

**`PIPELINE parallel_summarize`**
Declares the fan-out pattern explicitly: `search_results` fans into per-source summarization, then collects. This is the alternative to a Python `asyncio.gather()` call buried in application code with no visibility into what ran.

**`LOG`**
File-based structured logger. Every significant event — workflow started, source summarized, report completed — writes to `logs/research.log`. No external logging infrastructure needed; the generator produces a zero-dependency Rust logger using `std::fs`.

---

## How to compile and run

```bash
# From the agicore repo root:
node core/compiler/dist/cli.js generate \
  path/to/agicore-examples/01-research-pipeline/research_pipeline.agi \
  --output path/to/output/research_pipeline

cd path/to/output/research_pipeline
npm install
cargo tauri dev
```

The app opens with a Topics list. Create a topic with a name and query string, then trigger the research workflow from the topic detail view. Sources and the final report appear in their respective views as the pipeline completes.

To run the generated Rust tests:

```bash
cd path/to/output/research_pipeline/src-tauri
cargo test
```
