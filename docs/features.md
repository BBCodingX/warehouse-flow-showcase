# Feature catalog

The system was built in numbered phases: the core WMS is **Phases 0–21**, and the
intelligence / automation / simulation track is **Phases 1–9 + a simulation platform**, delivered
in parallel. Full phase-by-phase history is in the private repo's `CHANGELOG.md`.

Surface size: **45 API controllers**, **60+ Vue screens**.

---

## Core warehouse operations

### Inbound

| Capability | Notes |
|---|---|
| Purchase orders | Create, receive against, track fulfilment |
| Receiving workspace | Scan-driven; accepted / damaged split, lot & expiry (FEFO) capture |
| Putaway | Task per receipt, source & destination scan, expiration-aware, movement reconciliation |
| Relabel / manual fallback | When a scan can't resolve, a guarded manual path with its own audit trail |

### Inventory

| Capability | Notes |
|---|---|
| Receive / adjust / transfer stock | All asynchronous (`202 Accepted`), all through the ledger |
| Physical / Reserved / Available | Three quantities kept strictly distinct, enforced by check constraints |
| Stock buckets | `Available` / `Quarantine` / `Damaged` / `InspectionRequired` |
| Standard reason codes | Every movement is categorised |
| Inter-location & inter-warehouse transfers | Atomic, ledgered on both sides |
| Low-stock detection | `StockLowDetected` event, per-item threshold management |
| Blind cycle counts | Snapshot + stale-detection + reserved-stock guard + atomic reconciliation (ADR-027) |
| Multiple barcodes per product | Global barcode uniqueness |

### Orders → shipment

| Stage | Capability |
|---|---|
| Order | Create, line items, `PendingReservation` → `Reserved` / `ReservationFailed`, cancellation (releases reservation + picking task in one transaction) |
| Reservation | Multi-location, deterministic, all-or-nothing (ADR-028); idempotent; concurrency-safe; retry endpoint |
| Picking | Auto-created on `StockReserved`; task + wave picking; location-guided; operator ownership; short-pick returns the un-picked quantity to stock |
| Packing | Multiple packages per order, package↔carton binding, over-pack prevention, remaining-quantity view |
| Shipment | State machine `Created → Prepared → Dispatched → Delivered / DeliveryFailed / Cancelled`; status history with actor; atomic physical+reserved decrement at dispatch |
| Documents | Printable packing list & dispatch note as print-ready HTML (no PDF engine) |

### Returns

Customer returns with disposition back into the correct stock bucket.

### Logistic units

Pallet / tote / carton with barcoded identity, pallet nesting, atomic location movement, and full
lifecycle + movement history. `PickingTask–Tote` and `Package–Carton` links bind picked/packed
quantities to the physical carrier **without a second stock ledger**.

### Scanning (Phase 21)

One scan API that unambiguously resolves a product SKU/barcode, a warehouse-scoped location code,
or a logistic-unit code. Wired into receiving, transfer, tote assignment, guided picking, carton
packing, and blind cycle count. A wrong type / product / location / carrier scan **never** changes
an operation quantity. USB/manual reader plus an optional permissioned camera component.

---

## Reliability & operations surface

| Feature | What it does |
|---|---|
| Async operation tracking | Durable `Accepted → Published → Succeeded / Failed` per command, with human-readable operation numbers and a dedicated screen |
| Operational-exception queue | Assignment, history, resolution; auto-created only on terminal failure of the real retry chain |
| Personal notification feed | A user sees & dismisses their own background-operation failures; nav badge |
| DLQ management | Admin-only listing and redrive of dead-lettered messages |
| System status | `live` / `ready` / `degraded` health, worker heartbeats, dependency state |
| Backup / restore | Certified `pg_dump` custom-format + checksum + `pg_restore` into a fresh DB; overwrite of the live DB is refused |
| Data-retention sweep | Daily bounded-batch prune of processed outbox/inbox rows, completed async operations, aged audit events — the inventory ledger is never touched |
| Bulk CSV import | Products, locations, opening stock — dry-run by default, all-or-nothing commit, per-row error report |
| CSV management reports | On-hand, low stock, stock movements, cycle-count variance |
| Manager KPI strip | Fill rate, dock-to-stock time, open workload, weekly throughput |
| First-run setup wizard | Admin password → organization → first warehouse → module selection |

---

## Identity & security

| Feature | Notes |
|---|---|
| Local JWT auth | 15-min access token, memory-only (never in Web Storage); session restored from HttpOnly refresh cookie |
| Refresh-token rotation | SHA-256 hash stored, rotated every use, whole family revoked on reuse detection |
| Password hashing | PBKDF2-HMAC-SHA256, 600,000 iterations, transparent rehash on login |
| Account lockout + login rate limiting | Bounded lockout |
| Policy-based RBAC | One authorization policy per operation, five roles (Admin, WarehouseManager, WarehouseOperator, ShipmentOperator, Viewer) |
| Per-warehouse scoping | Enforced at the API, not just hidden in the UI |
| Admin user lifecycle | User creation, role/activation/password management; last active Admin cannot be demoted or deactivated |
| Transactional audit trail | Allowlisted entity mutations write an append-only `audit_events` row in the same transaction; secrets & tokens never captured |

Authorization matrix and configuration key list: [engineering.md](engineering.md#security).

---

## Intelligence, automation & simulation

A parallel track, governed by hard invariants (see [domain-model.md](domain-model.md#invariants)) —
none of it may become a source of truth or bypass domain commands.

| Phase | Layer | Content |
|---|---|---|
| **P1** | Warehouse Intelligence | Product/location health signals, insight framework with stateful lifecycle (`Active → Acknowledged/Resolved → Reopened`), trend & anomaly detection with sample-size + absolute-value guards, analysis runner + worker, API + UI |
| **P1.5** | Simulation & Digital Twin | Deterministic actors, virtual clock, demand/inbound generators, daily orchestrator, assertion engine, independent physical twin + reconciliation, `NormalDay` / `HighShortPick` / `HighDamage` scenarios, 30 / 90 / 365-day runs, CI integration |
| **P2** | Replenishment | Need calculation, task execution, policy management, intelligence signals |
| **P3** | Spatial slotting | Spatial model, slot-scoring engine with persisted evidence, recommendation workflow |
| **P4** | Route & wave optimization | Warehouse graph, route algorithms with fallback safety, wave composition, KPI measurement |
| **P5** | Mobile / RF | Mobile UI, online scan workflows, offline cycle count, sync reconciliation |
| **P6** | Labor intelligence | Task-duration tracking, productivity metrics, capacity-risk calculation |
| **P7** | Predictive inventory | Demand history, baseline forecast engine, backtesting, stockout-risk — trained only on the history WarehouseFlow actually recorded (no train/test leakage, INV-015) |
| **P8** | AI Copilot | Deterministic **read-only** tool surface, query router, explanation layer, authorization scoping, safety testing — every factual claim grounded in persisted evidence, never invented |
| **P9** | Approval-gated automation | Recommendation models, proposal validation, approval workflow, execution **through existing domain commands**, audit / idempotency / kill switch |

---

## Screen inventory (Vue)

<details>
<summary>60+ screens, by area</summary>

**Auth & shell** — Login, Account, Setup Wizard, Dashboard, System Status, Forbidden, Not Found

**Products & warehouses** — Product list / detail / form, Warehouse list / detail

**Inventory** — Inventory, Inventory Operations, Inventory Transactions, Warehouse Transfers,
Cycle Counts

**Inbound** — Purchase Orders, Inbound Planning, Receiving Workspace, Receipt list / detail,
Putaway Tasks + detail, Relabel Tasks, Movement Reconciliation

**Orders → shipment** — Order create / list / detail, Picking Task list / detail, Picking Waves,
Package list / detail, Shipment list / detail, Customer Returns

**Logistic units** — Logistic Units list / detail

**Operations** — Async Operations, My Operations, Operational Exceptions + detail,
Admin Operations, Operations Control, Reports, Data Import, Configuration

**Intelligence & automation** — Intelligence Dashboard, Intelligence Location / Product detail,
Replenishment Policies / Tasks + detail, Slotting, Route & Wave Optimization, Workforce Planning,
Copilot, Automation, Simulation Control Center

**Mobile** — a dedicated `mobile/` set for RF workflows

</details>
