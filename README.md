# 🔌 SAP B1 ↔ Odoo Integration — Architecture & Patterns

> Production-tested architecture documentation for bidirectional synchronization between SAP Business One (HANA) and Odoo, orchestrated through N8N. This repository documents the **engineering approach**, **patterns**, and **lessons learned** — implementation details are anonymized to protect proprietary code.

---

## 🎯 The Problem

Two ERPs, two sources of truth, zero communication. Manual data entry between SAP B1 and Odoo was creating:

- Duplicate customer / vendor records
- Inventory mismatches between systems
- Delayed financial reconciliation
- Hours of manual ops work weekly

## ✅ The Solution

A bidirectional, event-driven integration layer that keeps both systems in sync with minimal latency, full traceability, and graceful failure handling.

---

## 🏗️ Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│                     │         │                     │
│   SAP Business One  │◄────────┤        N8N          ├────────►│        Odoo         │
│      (HANA)         │ via DI- │  (Orchestrator)     │ via XML-│                     │
│                     │ API/SQL │                     │  RPC    │                     │
└─────────────────────┘         └──────────┬──────────┘         └─────────────────────┘
                                            │
                                            ▼
                                ┌─────────────────────┐
                                │  Reconciliation DB  │
                                │  (audit + state)    │
                                └─────────────────────┘
```

### Why N8N?

- **Visual workflow builder** — non-developers can audit logic
- **Native error handling** with retry mechanisms
- **Webhook + scheduled triggers** in the same platform
- **Lightweight** vs. heavier alternatives (MuleSoft, Boomi)
- **Self-hosted** — keeps sensitive ERP data inside the perimeter

---

## 🧩 Integration Patterns Implemented

### 1. Idempotency by design

Every sync operation includes a deterministic **idempotency key** (typically `source_system + entity_type + source_id`). If the same event arrives twice (network retry, webhook duplicate), the target system rejects it gracefully instead of creating duplicates.

```
idempotency_key = sha256(f"{source}:{entity}:{external_id}:{updated_at}")
```

### 2. Conflict resolution: last-write-wins with timestamps

Both systems can modify the same entity. When conflicts arise, we resolve by comparing `updated_at` timestamps from both ERPs and applying last-write-wins, with a reconciliation log for human review of edge cases.

### 3. Reconciliation database

A separate small database tracks:
- Every sync attempt (success / failure / partial)
- Mapping table between SAP B1 IDs and Odoo IDs
- Error queue for manual review
- Daily snapshot for drift detection

### 4. Exponential backoff for retries

API failures don't immediately mean a problem with our logic. We retry with exponential backoff (1s → 2s → 4s → 8s → fail) before pushing to dead-letter queue.

### 5. Schema validation layer

Before pushing to either system, payloads pass through a validation step that catches:
- Required field violations
- Type mismatches
- Reference integrity (FK lookups)
- Business rule violations

This catches ~95% of errors *before* they hit the ERP, where errors are expensive to undo.

---

## 📊 Entities Synchronized

| Entity | Direction | Frequency | Volume |
|--------|-----------|-----------|--------|
| Business Partners (Customers / Vendors) | Bidirectional | Real-time + nightly reconcile | High |
| Products / Items | Bidirectional | Hourly + on-change | Medium |
| Sales Orders | SAP → Odoo | Real-time | High |
| Purchase Orders | Odoo → SAP | Real-time | Medium |
| Invoices | SAP → Odoo | Real-time | High |
| Inventory Movements | Bidirectional | Real-time | Very High |

---

## 🔍 Key Lessons Learned

### What worked

✅ **Start with one entity in one direction.** Don't try to sync everything at once. Pick the highest-pain entity (for us: Business Partners) and one direction. Get it bulletproof. Then add complexity.

✅ **Invest in observability from day one.** Logs, metrics, and a simple dashboard showing "what synced when" saves more time than any clever code.

✅ **Mapping tables > guessing.** Don't assume IDs will match. Keep an explicit mapping table between SAP B1's `CardCode` and Odoo's `res.partner.id`. Worth its weight in gold during edge cases.

✅ **Document the business rules with the engineering.** "Why does X sync only when Y is set?" should be in the code, not in someone's head.

### What didn't

❌ **Trying to sync historical data on day one.** We tried a "big bang" migration. It failed three times. Eventually we cut over to "sync going forward, reconcile backward asynchronously."

❌ **Trusting ERP timestamps blindly.** Both systems have timezone gotchas, and SAP B1's `UpdateDate` and Odoo's `write_date` aren't perfectly aligned. We added a small tolerance window.

❌ **Webhook-only triggers.** Both systems have unreliable webhooks. We use webhooks for low-latency syncs and scheduled scans as the safety net.

---

## 🛠️ Tech Stack

- **Orchestration:** N8N (self-hosted)
- **SAP B1 access:** Service Layer (DI API) + direct SQL for read-heavy ops
- **Odoo access:** XML-RPC (preferred over REST for write operations — better error semantics)
- **State / Audit DB:** PostgreSQL
- **Monitoring:** N8N native + custom dashboards in Power BI
- **Secrets management:** Environment variables + N8N credentials store

---

## 📂 Repository Structure (planned)

```
sap-b1-odoo-integration/
├── README.md                     ← you are here
├── docs/
│   ├── architecture.md           ← deep dive
│   ├── entity-mappings.md        ← field-by-field mapping reference
│   ├── error-handling.md         ← retry & dead-letter patterns
│   └── reconciliation.md         ← drift detection approach
├── n8n-workflows/                ← anonymized workflow exports (JSON)
│   ├── partner-sync-saptoodoo.json
│   └── partner-sync-odootosap.json
├── sql-snippets/                 ← anonymized SAP B1 HANA queries
└── examples/                     ← sanitized payload examples
```

---

## 🤝 About the engineer

I'm **Roberto Iguardia** — Data Engineer specialized in ERP integration, ETL pipelines, and Power BI analytics. SAP B1 Super User certified.

- 🌐 [alejandroiguardia.netlify.app](https://alejandroiguardia.netlify.app/)
- 💼 [LinkedIn](https://www.linkedin.com/in/alejandro-iguardia-data-engineer)
- 📧 robertalerosales12@gmail.com

If you're building an SAP B1 integration and want to compare notes, I'm happy to chat.
