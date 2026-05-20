# Accelerando ERP/CRM

**A basic but extensible ERP + CRM for AI-native organizations.**

Compiles from a single `.agi` source file to a multi-tenant Axum web service, React frontend, and PostgreSQL database — ready to deploy on any Docker host or cloud provider.

Designed to be paired with the [Accelerando OIE](../accelerando-oie/). Together they form a complete AI-native business platform:

```
accelerando_erp.agi          accelerando_oie.agi
        │                            │
   Axum + React                Tauri desktop
   PostgreSQL                  SQLite (local)
        │                            │
        └──── oie_telemetry_out ─────┘
              TelemetryPacket
              (every action emits one)
```

**Two `.agi` files. One `docker-compose up`. Running.**

---

## What This Demonstrates

This app is the reference implementation for Agicore's **web target** — the same DSL that compiles to a desktop Tauri app can compile to a production-ready SaaS web service by adding three declarations:

```
TARGET web { RUNTIME axum  FRONTEND react  DEPLOY docker }
AUTH   jwt { STRATEGY bearer  EXPIRY 3600  REFRESH 86400 }
TENANT rows { ISOLATE tenant_id }
```

Everything else — entities, actions, views, stages, packets, channels — is 100% target-agnostic.

---

## Architecture

```
accelerando_erp.agi
│
├── TARGET / AUTH / TENANT      → Axum server, JWT middleware, tenant isolation
│
├── ENTITY × 10                 → PostgreSQL tables (tenant_id on every row)
│   Customer, Contact           → CRM: account and contact management
│   Vendor, Product, Employee   → ERP: supplier, catalog, headcount
│   Quote, QuoteLineItem        → Sales pipeline
│   Invoice, InvoiceLineItem    → Billing and receivables
│   ServiceTicket               → Support triage
│
├── STAGES × 3                  → Declarative state machines
│   Quote.status                → draft → sent → accepted / rejected / expired
│   Invoice.status              → draft → submitted → approved → paid / void
│   ServiceTicket.status        → new → in_progress → escalated → resolved → closed
│
├── PACKET TelemetryPacket      → Typed event shape (matches OIE expectation)
├── CHANNEL oie_telemetry_out   → Outbound stream to OIE
│
├── ACTION × 9                  → Business operations with EMIT telemetry
│   CreateInvoice               → AI-assisted invoice drafting
│   ApproveInvoice              → IMPL: status transition + approver record
│   MarkInvoicePaid             → IMPL: payment close-out
│   CreateQuote                 → AI-assisted quote introduction
│   ConvertQuoteToInvoice       → IMPL: quote-to-cash automation
│   CreateServiceRequest        → IMPL: ticket creation + auto-assign
│   UpdateTicketStatus          → IMPL: STAGES-validated transition
│   EscalateToTier2             → IMPL: tier-2 notification
│   ResolveServiceTicket        → AI-assisted resolution note
│
├── VIEW × 10                   → React pages (table, dashboard, kanban, split)
├── PREFERENCE × 3              → Currency, payment terms, support address
└── SEED × 9                    → Demo data matching OIE insight scenarios
```

---

## Generated Output

```
accelerando-erp/
├── Cargo.toml                  # axum 0.7, sqlx/postgres, tokio, tower-http, jsonwebtoken
├── src/
│   ├── main.rs                 # tokio::main, PgPool, CorsLayer, sqlx::migrate!
│   ├── db.rs                   # database pool creation
│   ├── error.rs                # AppError → HTTP status codes
│   ├── auth.rs                 # JWT claims, Bearer extraction, token generation
│   └── routes/
│       ├── customer.rs         # CRUD + tenant isolation from JWT claims
│       ├── contact.rs
│       ├── vendor.rs
│       ├── product.rs
│       ├── employee.rs
│       ├── quote.rs
│       ├── quote_line_item.rs
│       ├── invoice.rs
│       ├── invoice_line_item.rs
│       └── service_ticket.rs
├── migrations/
│   └── 0001_initial.sql        # PostgreSQL schema, UUID PKs, tenant_id indexes
├── src-frontend/               # React + TypeScript
│   └── src/
│       ├── client.ts           # fetch-based API client with JWT auth
│       └── api/                # typed per-entity modules
├── Dockerfile                  # multi-stage: rust builder → node builder → slim runtime
├── docker-compose.yml          # app + postgres:16-alpine + pgdata volume
├── .env.example                # DATABASE_URL, JWT_SECRET, PORT, CORS_ORIGIN
└── scaffold/                   # documentation
```

---

## Tenant Isolation

Tenant isolation is **structural**, not policy-based. Every generated Axum handler extracts `tenant_id` from the JWT claims — not from the request body — and injects it into every SQL query:

```rust
// generated — handler cannot be called without valid JWT
async fn list_invoices(
    auth: AuthClaims,   // tenant_id comes from JWT, not caller
    State(pool): State<PgPool>,
    Query(params): Query<ListParams>,
) -> Result<Json<Vec<Invoice>>, AppError> {
    let rows = sqlx::query_as!(
        Invoice,
        "SELECT * FROM invoices WHERE tenant_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3",
        auth.tenant_id,   // structural — no code path skips this
        params.limit.unwrap_or(50),
        params.offset.unwrap_or(0)
    )
    .fetch_all(&pool)
    .await?;
    Ok(Json(rows))
}
```

No query exists without the tenant filter. It cannot be forgotten.

---

## Telemetry Connection

Every action emits a `TelemetryPacket` to `oie_telemetry_out`. The OIE's `telemetry_ingress` channel receives the same packet type. The packet schema is identical in both apps — declared independently, matched by convention.

The OIE's three demo insights are drawn from exactly these event types:

| OIE Insight | Driving ERP Events |
|---|---|
| "Invoice approval averaging 4.2 days" | `invoice_created`, `invoice_approved` |
| "Quote-to-cash handoff is manual in 67% of cases" | `quote_created`, `quote_converted` |
| "Support escalation rate up 40% this week" | `service_request_created`, `ticket_escalated` |

Run both apps in demo mode and the OIE dashboard immediately has data to reason over.

---

## Running Together

```yaml
# docker-compose.accelerando.yml (at apps/ level)
services:
  erp:
    build: ./accelerando-erp
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/accelerando_erp
      JWT_SECRET: your-secret-here
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: accelerando_erp
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  # OIE runs as a Tauri desktop app alongside
  # — it connects to the same telemetry channel

volumes:
  pgdata:
```

```bash
# Compile both apps from source
agicore compile accelerando_erp.agi --out ./accelerando-erp/
agicore compile accelerando_oie.agi --out ./accelerando-oie/

# Start the web stack
docker-compose -f apps/docker-compose.accelerando.yml up

# Launch the OIE desktop app
cd apps/accelerando-oie && cargo tauri dev
```

---

## Extending the ERP

Each business module follows the same pattern — one ENTITY per concept, STAGES for workflow, VIEW for the UI, ACTION for business operations.

### Adding Purchase Orders

```
ENTITY PurchaseOrder {
  tenant_id:   string REQUIRED
  vendor_id:   string REQUIRED
  total:       float  REQUIRED
  status:      string = "draft"
  approved_by: string
  TIMESTAMPS
  CRUD create, read, list, edit
}

STAGES PurchaseOrder.status {
  TRANSITION "draft" -> "submitted" {
    MATCH any
    REQUIRE PurchaseOrder.vendor_id IS NOT NULL
    REQUIRE PurchaseOrder.total > 0
  }
  TRANSITION "submitted" -> "approved" {
    MATCH any
    REQUIRE PurchaseOrder.approved_by IS NOT NULL
  }
}

ACTION ApprovePurchaseOrder {
  INPUT   po_id: string, approver_id: string
  OUTPUT  approved: bool
  IMPL    "validate approval authority, transition PO to approved"
  EMIT po_approved {
    source: string, event_type: string, tenant_id: string, success: bool
  }
}
```

Recompile. PostgreSQL migration runs. Axum routes appear. React page renders. OIE picks up `po_approved` events automatically.

---

## Seed Data and the OIE Demo

The nine seed rows create business context that matches the OIE's demo insights:

- Three customers (Acme, Brightline, Vertex) at different tiers
- Three products (Starter, Professional, Implementation Services)
- Two employees (Sales Manager, Support Lead) for assignment workflows
- One vendor (Northgate Supplies)

Open the OIE in demo mode and the insight "Invoice approval averaging 4.2 days" references activity that traces back to these customers and these invoice workflows.

The full Accelerando stack is coherent end-to-end — not a collection of disconnected demos.
