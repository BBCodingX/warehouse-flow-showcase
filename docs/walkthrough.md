# Visual walkthrough

A guided tour of the goods flow and the messaging that drives it.

> Live UI screenshots will be added here later. Everything below is generated from the project's
> own design diagrams.

## The goods flow

```mermaid
flowchart LR
    PO[Purchase order] --> REC[Receiving<br/>accepted / damaged, lot + expiry]
    REC --> PUT[Putaway<br/>scan source → destination]
    PUT --> INV[(Inventory<br/>physical / reserved / available)]
    INV --> RES[Reservation<br/>multi-location, all-or-nothing]
    RES --> PICK[Picking<br/>task or wave, guided]
    PICK --> PACK[Packing<br/>package ↔ carton]
    PACK --> SHIP[Shipment<br/>Created → Prepared → Dispatched → Delivered]
    INV -. cycle count .-> INV
    INV -. transfer .-> INV
    SHIP -. customer return .-> INV
```

Every arrow that changes stock goes through the append-only ledger and is attributed to the
initiating operator.

## How a stock command travels

```mermaid
graph LR
    API[ASP.NET Core API] -->|Transactional Outbox<br/>same transaction| PG[(PostgreSQL<br/>outbox_messages)]
    IW[Inventory Worker] -->|Transactional Outbox| PG
    OW[Order + Reservation Worker] -->|Transactional Outbox| PG
    SW[Shipment Worker] -->|Transactional Outbox| PG

    PG -->|Outbox Publisher<br/>polling, batched, SKIP LOCKED| MT[MassTransit publish]
    MT --> ExReserve[reserve-stock exchange]
    MT --> ExReceive[receive-stock exchange]
    MT --> ExTransfer[transfer-stock exchange]
    MT --> ExPicking[create-picking-task exchange]

    ExReceive --> QReceive[receive-stock queue] --> IW
    ExTransfer --> QTransfer[transfer-stock queue] --> IW
    ExReserve --> QReserve[reserve-stock queue] --> OW
    ExPicking --> QPick[create-picking-task queue] --> SW

    QReserve -.retry exhausted.-> DLQ[reserve-stock_error]
    QReceive -.retry exhausted.-> DLQ
    DLQ --> Admin[DLQ admin screen + redrive]
```

The API never publishes directly. It writes the message to `outbox_messages` in the same
transaction as the domain change; a polling publisher drains the table to RabbitMQ and only then
marks the row processed. See [reliability.md](reliability.md#1-the-dual-write-problem--transactional-outbox).

## Order → reservation → picking

```mermaid
sequenceDiagram
    participant FE as Vue Frontend
    participant API as ASP.NET Core API
    participant PG as PostgreSQL
    participant MQ as RabbitMQ
    participant OW as Order + Reservation Worker
    participant SW as Shipment Worker

    FE->>API: POST /api/orders
    API->>PG: Insert Order (PendingReservation) + OutboxMessage (OrderCreated, ReserveStock)
    API-->>FE: 201 Created (orderId, PendingReservation)
    API->>MQ: Outbox Publisher (polling) publishes
    MQ->>OW: ReserveStock command

    OW->>PG: idempotency_key check (stock_reservations)
    alt already processed
        OW->>MQ: no-op ack
    else new request
        OW->>PG: select/lock Available inventory_items, deterministic order
        alt enough stock
            OW->>PG: allocate across locations + reserved_quantity update
            alt optimistic retry budget exhausted
                OW->>PG: SELECT ... FOR UPDATE (ADR-017)
            end
            OW->>PG: OutboxMessage (StockReserved, CreatePickingTask) + Order.Status = Reserved
        else not enough
            OW->>PG: OutboxMessage (StockReservationFailed) + Order.Status = ReservationFailed
            Note over OW,PG: no CreatePickingTask (ADR-020)
        end
        OW->>MQ: Outbox Publisher publishes the outcome
    end

    MQ->>SW: CreatePickingTask (only on StockReserved)
    SW->>PG: picking_tasks(order_id) partial unique index check (idempotent)
    SW->>PG: PickingTask insert + OutboxMessage (PickingTaskCreated)

    FE->>API: GET /api/orders/{id} (polling)
    API-->>FE: resolved status (Reserved / ReservationFailed)
```

## Stock transfer (single transaction, both sides)

```mermaid
sequenceDiagram
    participant FE as Vue Frontend
    participant API as ASP.NET Core API
    participant PG as PostgreSQL
    participant MQ as RabbitMQ
    participant IW as Inventory Worker

    FE->>API: POST /api/inventory/transfer (TransferStock)
    API->>PG: OutboxMessage (TransferStock command)
    API-->>FE: 202 Accepted
    API->>MQ: Outbox Publisher (polling, batch) publishes
    MQ->>IW: TransferStock command

    IW->>PG: processed_messages check (idempotency)
    alt enough stock
        IW->>PG: source physical_quantity down (ledger: TransferOut)
        IW->>PG: destination physical_quantity up (ledger: TransferIn)
        IW->>PG: OutboxMessage (StockTransferred) + processed_messages row
        Note over IW,PG: one transaction
    else not enough
        IW->>PG: rejected, logged
    end
```

## Shipment state machine

```mermaid
stateDiagram-v2
    [*] --> Created: CreateShipment
    Created --> Prepared: PrepareShipment
    Prepared --> Dispatched: DispatchShipment
    Dispatched --> Delivered: MarkShipmentAsDelivered
    Dispatched --> DeliveryFailed: RecordDeliveryFailure
    Created --> Cancelled: cancel (Created only)
    Delivered --> [*]
    DeliveryFailed --> [*]
    Cancelled --> [*]
```

At dispatch, physical **and** reserved quantities are decremented atomically, and the ledger
records a `ShipmentDispatch` movement referencing the order. Only a `Created` shipment can be
cancelled — later stages carry physical hand-off risk.

## Async operation lifecycle

```mermaid
stateDiagram-v2
    [*] --> Accepted: command written to Outbox (same txn as AsyncOperation row)
    Accepted --> Published: broker publish confirmed
    Published --> Succeeded: Inbox + domain transaction committed
    Published --> Failed: central retry policy exhausted
    Failed --> [*]: one OperationalException created + operator notified
    Succeeded --> [*]
```

`202 Accepted` only proves the request reached the Outbox — this lifecycle makes the real outcome
of background work visible. See [reliability.md](reliability.md#5-failures-that-must-not-vanish--durable-async-lifecycle--dlq).
