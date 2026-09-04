<div align="center">

# WarehouseFlow

**An event-driven Warehouse Management System — receiving → putaway → inventory → reservation → picking → packing → shipping — that keeps stock correct under concurrency.**

`.NET 10` · `Vue 3` · `PostgreSQL 17` · `RabbitMQ` · `Redis` · `Docker Compose`

### 📄 [Read the walkthrough →](https://YOUR-GITHUB-USERNAME.github.io/warehouse-flow/)

*(replace with your GitHub Pages URL once deployed — see [SETUP.md](SETUP.md))*

</div>

---

> **This is a showcase repository.** WarehouseFlow is a personal portfolio project, not a product —
> the application source is private. What's here is the design: the architecture, the
> distributed-systems patterns and *why* they were chosen, the full feature surface, the domain
> model, and the testing strategy.
>
> Happy to demo the running system or go through the code in an interview.

## Start here

- **[`index.html`](index.html)** — the visual walkthrough (also the GitHub Pages site). A 2–3
  minute scroll: architecture, the hard problems it solves, a screen recording of an order being
  placed → reserved → turned into pick work, screenshots of the running system, the digital-twin
  test gate, the feature surface, and the engineering practices.

## Deeper reading

| Document | Contents |
|---|---|
| [docs/architecture.md](docs/architecture.md) | System & container diagrams, the 11 backend projects, sync vs async, failure modes |
| [docs/reliability.md](docs/reliability.md) | Transactional Outbox / Inbox, the concurrency ladder, multi-location reservation, DLQ, cycle-count safety — with real code |
| [docs/walkthrough.md](docs/walkthrough.md) | Sequence & state diagrams for every key flow |
| [docs/features.md](docs/features.md) | 45 API controllers, 60+ screens, phase-by-phase |
| [docs/domain-model.md](docs/domain-model.md) | ER diagram, the three stock quantities, 16 invariants |
| [docs/engineering.md](docs/engineering.md) | Test strategy (1,300+ tests), authorization matrix, secret handling, ADR summary |

## The short version

| | |
|---|---|
| Tests | **1,300+** automated tests across backend, simulator, frontend & E2E — 791 backend tests run against real PostgreSQL & RabbitMQ (Testcontainers) |
| Backend | ~44k lines of C# · API + 4 worker services · 13 MassTransit consumers · 30 ADRs |
| Frontend | ~24k lines of Vue 3 + TypeScript · 60+ screens · Vitest + Playwright E2E |
| Simulator | ~14k lines · deterministic 30 / 90 / 365-day digital-twin runs with reconciliation + invariant assertions, gated in CI |
| Docs | ~107k words · 30 ADRs · a phased roadmap |
| External runtime dependencies | **0** — no SMTP, SMS, push, maps, GPS, payment, or third-party AI |

Runs entirely on one machine:

```bash
cp .env.example .env
docker compose up --build      # API + 4 workers + Postgres + RabbitMQ + Redis + Vue SPA
```

---

<sub>Personal portfolio project. Application source private; documentation shared for review only. © 2026, all rights reserved.</sub>
