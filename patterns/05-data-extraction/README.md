# Example 05 — Data Extraction

Extracts structured data from unstructured documents (contracts, invoices, emails, receipts) using typed ENTITY schemas as the extraction target and a three-stage pipeline to classify, extract, and validate.

---

## What problem this solves

Document processing pipelines extract structured data from unstructured text — but the extracted values often arrive as raw strings that are never validated against a schema before being stored. A date field receives "January 15th, 2026" and "01/15/26" and "2026-01-15" interchangeably; a number field receives "$2,500.00" when the database expects `2500.0`. This example makes the schema explicit: each extracted field is stored as an `ExtractedRecord` entity with a confidence score, the extraction run is tracked, and a Rust validation action checks the extracted JSON against the expected schema before committing.

---

## The agentic anti-pattern it replaces

Most extraction pipelines call an LLM with a JSON schema in the prompt and then call `json.loads()` on the response, crossing their fingers. There is no confidence tracking, no validation of field types against a declared schema, and no record of which model ran the extraction or how many fields it found.

---

## Key Agicore declarations

**`TYPE DocumentKind`, `ExtractionStatus`** — union type aliases ensuring documents can only have valid kind and status values.

**`ENTITY ExtractedRecord`** — every extracted field is a first-class row in SQLite with a confidence score. This makes the extraction auditable: you can query which fields had confidence below a threshold and flag them for human review.

**`PIPELINE extraction_pipeline`** — three stages with explicit data flow. The `classify` stage output flows into the `extract` stage input via a `CONNECTION` declaration. The connections are the type boundary — data flows only along declared paths.

**`ACTION validate_extraction`** — uses `IMPL` to generate a protected Rust stub. Custom validation logic (type coercion, range checks, format normalization) lives in Rust and is never overwritten by the compiler.

---

## How to compile and run

```bash
node core/compiler/dist/cli.js generate \
  path/to/agicore-examples/05-data-extraction/data_extraction.agi \
  --output path/to/output/data_extraction

cd path/to/output/data_extraction
npm install
cargo tauri dev
```

After generating, implement the validation logic in `src-tauri/src/commands/validate_extraction.rs` (the file is `// @agicore-protected` and will not be overwritten).
