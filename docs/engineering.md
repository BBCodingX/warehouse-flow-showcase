# Engineering practices

## Testing strategy

Tests run entirely locally on free tooling. No external or paid test service is used anywhere.

| Layer | Tooling | Coverage |
|---|---|---|
| **Domain unit** | xUnit | Aggregate invariants — negative stock impossible, `reserved ≤ physical`, illegal state transitions rejected |
| **Application** | xUnit + mocks | Command handler / service business rules |
| **API integration** | `WebApplicationFactory` + **Testcontainers PostgreSQL 17** | Endpoints end-to-end on a real database; scan resolution, ledger v2, denied-attempt audit, safe shipment cancellation |
| **Messaging integration** | Testcontainers RabbitMQ (management) | Transactional Inbox, delivery/redelivery, error queue, retry → DLQ → redrive, Outbox resilience |
| **Concurrency** | Parallel tasks against real PostgreSQL | Simultaneous reservations never exceed available stock; optimistic conflict → bounded retry; `FOR UPDATE` fallback path; check-constraint rejection |
| **Idempotency** | xUnit | Duplicate `StockReserved` → one `CreatePickingTask`; duplicate publish → one domain effect via `processed_messages`; failed reservation → no picking task |
| **Frontend unit** | Vitest + Vue Test Utils | Component validation, list render, status badges, barcode bounce/busy behaviour, polling stop conditions |
| **E2E** | Playwright (Chromium) | Protected routes, Viewer mutation invisibility, session expiry, unauthorized mutation produces no side-effect, full `Delivered` flow, Admin user lifecycle, PO → receiving → putaway scan chain — around a dozen scenarios |
| **Release certification** | Isolated Docker Compose | `pg_dump`/`pg_restore` parity, migration/constraint parity, k6 load + state-invariant gate, Redis degraded mode, RabbitMQ outbox buffering/drain, PostgreSQL outage/recovery |
| **Simulator** | Deterministic runs in CI | 30 / 90 / 365-day scenarios with mandatory reconciliation assertions between the physical twin and WarehouseFlow state |

Current tally: **~1,300 automated tests** on a full run — 791 backend (xUnit, real infrastructure),
252 simulator, ~140 frontend unit (Vitest), plus the Playwright E2E suite. Frontend typecheck and
production build are additional quality gates. `./scripts/verify-release.sh` runs every gate in one
command.

## Security

### Authorization matrix

| Operation | Admin | WarehouseManager | WarehouseOperator | ShipmentOperator | Viewer |
|---|:-:|:-:|:-:|:-:|:-:|
| View products / stock | ✓ | ✓ | ✓ | ✓ | ✓ |
| Create / update products | ✓ | ✓ | — | — | — |
| Create warehouses / locations | ✓ | ✓ | — | — | — |
| Receive stock | ✓ | ✓ | ✓ | — | — |
| Adjust stock | ✓ | ✓ | — | — | — |
| Transfer stock | ✓ | ✓ | ✓ | — | — |
| Create orders | ✓ | ✓ | ✓ | — | — |
| Picking | ✓ | ✓ | ✓ | — | — |
| Packing | ✓ | ✓ | — | ✓ | — |
| Create / update shipments | ✓ | ✓ | — | ✓ | — |
| View / redrive DLQ | ✓ | — | — | — | — |
| View audit events | ✓ | — | — | — | — |

On top of role policies, every warehouse-bound query and mutation is **scoped to the warehouses
the user may access**, enforced in the API layer.

### Secret handling

- No real credential exists in the repo or the docs — only placeholders
  (`<POSTGRES_PASSWORD>`, `<JWT_SECRET>`, `<INITIAL_ADMIN_PASSWORD>`, …).
- `.env`, `appsettings.Local.json`, `secrets.json`, PEM / PKCS#12 files are git-ignored; only
  placeholder `.env.example` is committed.
- The recommended local flow (`scripts/run-local-secure.sh`) generates a fresh cryptographic
  random value per service into a `umask 077` temp file, and deletes the file + all ephemeral
  volumes on Ctrl+C.
- Logs never carry a connection string, password, JWT secret, or token — structured log templates
  are checked for this.
- Production defaults require HTTPS and a `Secure` refresh cookie; local Compose disables only
  those two, explicitly, for `localhost` HTTP.
- `tests/security/verify-repository-security.mjs` audits source + build output for high-entropy
  secret signatures.
- The initial admin password must be ≥ 12 chars and cannot be a placeholder — the API fail-fasts
  otherwise.

## Architecture Decision Records

30 ADRs record every significant choice with context, alternatives considered, consequences, and
risks. Highlights:

| ADR | Decision | Why it's interesting |
|---|---|---|
| ADR-004 | MassTransit **8.x**, not 9.x | 9.x demands a paid runtime license key and the bus won't start without it — a direct conflict with the "no paid services" constraint. 8.x is Apache-2.0. |
| ADR-008 | Transactional Outbox | Solves the dual-write problem; accepts at-least-once delivery as the trade-off. |
| ADR-009 | `ProcessedMessage` Inbox | Idempotent consumers under at-least-once. |
| ADR-017 | Optimistic concurrency, `FOR UPDATE` as fallback | The layered ladder — no single mechanism is trusted for stock correctness. |
| ADR-018 | Polling Outbox publisher | `LISTEN/NOTIFY` deferred, not adopted — polling is simpler to debug and fast enough at this scale. |
| ADR-022 | Short JWT + rotating refresh token | Reuse detection revokes the whole family. |
| ADR-023 | Transactional audit trail | Audit rows commit or roll back with the domain change. |
| ADR-026 | Durable async command lifecycle | `202 Accepted` isn't proof of success — `AsyncOperation` makes background failure visible. |
| ADR-027 | Cycle-count reconciliation safety | A stale count can't overwrite newer stock movements. |
| ADR-028 | Multi-location all-or-nothing reservation | Uses total available stock without leaving orders unnecessarily unfulfilled. |
| ADR-030 | Bounded list pagination contract | `offset`/`limit`, max 200, deterministic order, `X-Total-Count`. |

The full set — plus a risk register, a definition-of-done, engineering rules, and a requirement
traceability matrix — lives under `docs/` and `docs/roadmap/` in the private repo.

## Coding standards (selected)

- Generic patterns (Generic Repository, MediatR, CQRS, Event Sourcing, Saga) are used **only** when
  they solve a concrete problem here — none is adopted by default.
- One PostgreSQL instance, shared by all services — no database-per-service theatre at this scale.
- Worker count is deliberately bounded; there is no Notification / Mail / SMS / Location / Tracking
  worker because there is no such feature.
- Turkish operator-facing copy, English code and documentation of record.
