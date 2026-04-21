# Session 03 — Pitfall Cards (Modular)

> Each card is self-contained. Pick any card and insert it into the relevant slide section.
> The running scenario is **Tradr**, a B2C marketplace with Catalog / Pricing / Inventory / Shipping / Finance services.

---

## Index

| Card | Pitfall | Insert near |
|---|---|---|
| P-00 | "Independent tables" illusion | Slide 6 |
| P-01 | "We'll split the DB later" | Slide 8 |
| P-02 | Cross-service foreign keys | Slide 5 |
| P-03 | "Who owns `status`?" | Slide 7 |
| P-04 | Master table no one can own | Slide 7 / Exercise 1 |
| P-05 | Dual-write | Slide 10 |
| P-06 | Publishing before commit | Slide 10–11 |
| P-07 | Outbox without idempotent consumer | Slide 12 |
| P-08 | Out-of-order delivery | Slide 16 |
| P-09 | Shared read replica "just for reads" | Slide 17 |
| P-10 | No projection rebuild strategy | Slide 16 |

---

## ACT 1 — Ownership

---

### P-00 — The "Independent Tables" Illusion

**Tradr did:**
Split their monolith into `Order`, `Inventory`, and `Billing` services. They kept them all pointing to the same database but created strict rules: "Each service only writes to its own table." 

**Why it felt right:**
Table separation feels like decoupling. They didn't use `JOIN`s or foreign keys, so they assumed they had successfully built microservices.

**What broke:**
The checkout API wrapped the entire sequence (`create_order`, `reserve_stock`, `charge_account`) in a single `BEGIN TRANSACTION`. When the Billing table locked up during a batch job, the Inventory updates also blocked, and the `reserve_stock` calls timed out. The system was still completely coupled at the data layer and failed as a distributed monolith.

**The pattern:**
True independence requires independent transaction boundaries, not just separate table names. If three services rely on one database connection to succeed together, they are a monolith.

**Fix:** Remove the global transaction. Use the Outbox Pattern or Sagas to coordinate multi-step workflows asynchronously, accepting eventual consistency.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

participant "API Gateway\n(Checkout)" as API
participant "Order\nService" as ORD
participant "Inventory\nService" as INV
participant "Billing\nService" as BILL

box "Shared RDBMS" #LightCyan
database "orders" as DB_O
database "inventory_levels" as DB_I
database "payments" as DB_B
end box

group Global Transaction (Shared Connection / 2PC)
  API -> ORD : create_order()
  ORD -> DB_O : INSERT
  
  API -> INV : reserve_stock()
  INV -> DB_I : UPDATE
  
  API -> BILL : charge_account()
  BILL -> DB_B : INSERT
end

note over API, DB_B
  The compute is split, but they still act as one monolith
  tied together by a shared synchronous transaction.
end note
@enduml
```

---

### P-01 — "We'll Split the DB Later"

**Tradr did:**
Split service deployments first. Left the DB shared "temporarily" — the API endpoints were the urgent deliverable.

**Why it felt right:**
The DB split is risky, slow, and invisible to users. Product pressure made the service split the priority.

**What broke:**
The Inventory team added `reserved_quantity`. Finance's reporting query broke that same night — it used `SELECT *` from the products table. The next migration script required sign-off from three other teams before it could merge.

**The pattern:**
"Just for now" became "we've been meaning to do this for 18 months." The shared DB became the permanent integration layer.

**Fix:** The service extraction is not done until the schema is also extracted. DB-per-service is part of the boundary definition, not a follow-up task.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

node "Catalog Service"   as CAT
node "Pricing Service"   as PRICE
node "Inventory Service" as INV
node "Shipping Service"  as SHIP
node "Finance Service"   as FIN
database "Shared DB ⚠️" as DB

CAT   --> DB
PRICE --> DB
INV   --> DB
SHIP  --> DB
FIN   --> DB

note bottom of DB
  Services are deployed separately.
  DB schema is still shared.
  One migration can break all five.
end note
@enduml
```

---

### P-02 — Cross-Service Foreign Keys

**Tradr did:**
Left a foreign key in place: `inventory_levels.order_id REFERENCES orders.id` "for safety" while splitting the databases.

**Why it felt right:**
Referential integrity is a core RDB feature. They wanted to ensure an inventory reservation could never exist without a corresponding order row.

**What broke:**
During a Checkout flow, the Order was created (Tx 1) and Inventory was reserved (Tx 2). However, Billing failed (Tx 3). The Saga attempted to compensate by deleting the Order row, but the DB blocked the `DELETE` because the Inventory table still held a reference to that `order_id`. The compensation was blocked, leaving the system in a broken, inconsistent state.

**The pattern:**
A FK across service boundaries is a runtime dependency disguised as a safety feature. It gives the referencing service hidden veto power over the owning service's data lifecycle and recovery paths.

**Fix:** Reference by stored ID only. Referential safety becomes the owning service's responsibility via its API and business logic, not the DB engine's constraint.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

participant "API Gateway\n(Checkout)" as API
participant "Order\nService" as ORD
participant "Inventory\nService" as INV
participant "Billing\nService" as BILL

box "Shared RDBMS" #LightCyan
database "orders" as DB_O
database "inventory_levels" as DB_I
database "payments" as DB_B
end box

note over DB_I
  inventory_levels.order_id
  REFERENCES orders.id ⚠️
end note

group Tx 1: Order Service
  API -> ORD : create_order()
  ORD -> DB_O : INSERT ✅ COMMIT
end

group Tx 2: Inventory Service
  API -> INV : reserve_stock()
  INV -> DB_I : UPDATE (order_id=ORD-99) ✅ COMMIT
end

group Tx 3: Billing Service
  API -> BILL : charge_account()
  BILL -[#red]->x DB_B : INSERT ❌ FAIL
end

note over API : Saga: compensation required

group Compensation: Order Service
  API -> ORD : cancel_order()
  ORD -[#red]->x DB_O : DELETE FROM orders\nWHERE id='ORD-99' ❌ BLOCKED\n(inventory_levels FK still references it)
end

note over ORD
  Order cannot clean up its own table.
  Inventory's FK holds the veto.
end note
@enduml
```

---

### P-03 — "Who Owns `status`?"

**Tradr did:**
Kept `products.status` (active / inactive / discontinued) in the shared table. All five services read and sometimes wrote it.

**Why it felt right:**
"Status" seemed like a universal field — a single source of truth that everyone could trust.

**What broke:**
Three teams were updating the same column with three incompatible meanings:

| Service | What they meant by `status` |
|---|---|
| Catalog | Publication state: draft / published / archived |
| Inventory | Stock state: available / out-of-stock / backordered |
| Finance | Tax registration: taxable / exempt / pending-review |

The column became meaningless. A product "inactive" for Finance (pending tax review) was "active" for Catalog. Bugs became impossible to diagnose without knowing which team last wrote the field.

**The pattern:**
A field that means something different to each writer is not a shared field — it is multiple distinct concepts merged into one column with no owner.

**Fix:** Each service defines its own lifecycle state with a clear owner: `catalog_status`, `availability`, `tax_registration_status`. The vague shared `status` is retired.

---

### P-04 — Master Table No One Can Own

**Tradr did:**
Kept the `products` table as-is across all five services. Teams owned different column groups within the same table.

**Why it felt right:**
Splitting the table felt like a large, risky migration with no user-visible payoff. "We'll just be careful about which columns each team touches."

**What broke:**
With 60+ columns and no single owner, the table became a coordination tax:
- No team could add a column without reviewing with four others
- No team could drop a column — it might be in another team's `SELECT *`
- Inventory's batch-job load spikes caused latency for Catalog reads
- One bad migration script caused a 2-hour outage

**The pattern:**
A large shared master table with no business team owner is the wrong abstraction, not just the wrong location.

**Fix:** Apply the bounded-context split. Each service owns only the columns its business function writes. (→ See Exercise 1)

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

package "Before: one shared table" {
  database "products\n(60 columns, 5 writers)" as PT
}

package "After: each service owns its slice" {
  database "catalog_products\nname, description, images" as CAT_T
  database "pricing_rules\nlist_price, discount_rules"  as PRICE_T
  database "inventory_items\nsku, qty_on_hand"          as INV_T
  database "shipping_specs\nweight, dimensions"         as SHIP_T
  database "finance_products\ntax_class, cost_price"    as FIN_T
}

note bottom of PT
  No one can drop a column.
  No one can add without sign-off.
  Migrations require all-team review.
end note
@enduml
```

---

## ACT 2 — Messaging

---

### P-05 — The Dual-Write

**Tradr did:**
```python
def create_order(payload):
    order = db.insert("orders", payload)       # Step 1
    queue.publish("OrderCreated", order)        # Step 2
    return order
```

**Why it felt right:**
The natural sequence: persist, then notify. The obvious order of operations.

**What broke:**
The message broker had a 90-second network partition at 2 AM. Step 1 (DB write) succeeded. Step 2 (queue publish) failed silently. Inventory never received `OrderCreated`. 140 orders had no stock reservation the next morning.

A developer then reversed the order to "be safe." The event reached Inventory before the order row was committed. Inventory could not find the order row when it tried to validate.

**The pattern:**
Two independent systems share no transaction boundary. Any failure between the two steps leaves producer and consumers in disagreement.

**Fix:** Outbox pattern — write to `orders` and `outbox` in one ACID transaction. The relay delivers from a committed, consistent state.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

participant "Order Service"     as S
database    "Database"          as DB
participant "Message Broker"    as Q
participant "Inventory Service" as INV

S  -> DB : INSERT INTO orders ... ✅ committed
S  -[#red]->x Q  : publish OrderCreated ❌ network error

note over INV
  Never receives the event.
  Stock is not decremented.
end note

note over DB, Q
  DB and Inventory now disagree.
  No transaction spans both systems.
end note
@enduml
```

---

### P-06 — Publishing Before Commit

**Tradr did:**
```python
with db.transaction():
    order = db.insert("orders", payload)
    queue.publish("OrderCreated", order)   # ← inside the tx block
    db.commit()
```

**Why it felt right:**
"If I publish inside the transaction block, it's effectively atomic, right?"

**What broke:**
The queue publish succeeded. `db.commit()` then failed (disk full on the DB host). Inventory received and processed `OrderCreated` — and decremented stock. The order row was rolled back. Inventory's stock was decremented for an order that never existed.

**The pattern:**
Publishing inside a transaction block does not include the publish in the transaction. Queue publish is a side effect against an external system — it operates entirely outside the DB's ACID scope. The commit is the only guarantee boundary.

**Fix:** The outbox row IS the publication intent. The commit atomically persists both the business row and the outbox row. The relay runs outside the transaction and delivers from a safely committed state.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

participant "Order Service"     as S
database    "Database"          as DB
participant "Message Broker"    as Q
participant "Inventory Service" as INV

note over S, DB : BEGIN TRANSACTION

S -> DB : INSERT INTO orders ...
S -> Q  : publish OrderCreated ✅\n(external call — outside ACID)

Q  -> INV : deliver OrderCreated
INV -> INV : decrement stock ✅

S -[#red]->x DB : COMMIT ❌ (disk full — rolled back)

note over DB  : Order row rolled back
note over INV : Stock already decremented ❌\nOrder does not exist in DB
@enduml
```

---

### P-07 — Outbox Without Idempotent Consumer

**Tradr did:**
Implemented the outbox pattern correctly on the publisher side. Did not implement inbox deduplication on the Inventory consumer.

**Why it felt right:**
"We have at-least-once delivery. Duplicates will be rare in practice."

**What broke:**
The relay published `OrderCreated (msg-001)` to the broker. Inventory began processing — stock decremented. The relay crashed before marking the outbox row as delivered. On restart, it re-published the same message. Inventory received it a second time. No deduplication check → two stock decrements → negative stock on a high-demand item.

**The pattern:**
Outbox guarantees at-least-once delivery. At-least-once means duplicates are expected, not occasionally possible. The consumer must be idempotent by design, not by assumption.

**Fix:** Inbox pattern — before processing, check the `inbox` table for `message_id`. If already present, skip. Insert `message_id` and process in one local transaction.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

participant "Relay"             as R
participant "Message Broker"    as Q
participant "Inventory Service" as INV
database    "Inventory DB"      as DB

R   -> Q   : publish OrderCreated (msg-001) ✅
Q   -> INV : deliver msg-001
INV -> DB  : decrement stock ✅

note over R : Relay crashes before\nmarking outbox row delivered

R   -> R   : restart
R   -> Q   : re-publish OrderCreated (msg-001) ✅
Q   -> INV : deliver msg-001 again
note over INV : No inbox check →\nprocesses duplicate
INV -> DB  : decrement stock again ❌\n(negative stock)
@enduml
```

---

### P-08 — Out-of-Order Delivery Wrecks the Projection

**Tradr did:**
The Search service subscribed to `StockChanged` events and applied each event to its `product_search_index` in the order received.

**Why it felt right:**
"Events arrive in order — it's a queue."

**What broke:**
Two `StockChanged` events produced seconds apart. A transient processing error caused the broker to requeue the older one. The newer event (v=6, `in_stock=false`) arrived first and was applied correctly. The older event (v=5, `in_stock=true`) arrived late and silently overwrote the newer state. The product displayed as in-stock even though stock was actually zero.

**The pattern:**
Message brokers guarantee ordering only under specific conditions (single partition, no retries). Requeues and retry policies break delivery order in practice.

**Fix:** Include a `version` or `event_sequence` in every event. Before writing to the projection, guard with: apply only if `incoming_version > current_version`. Silently discard stale events.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

participant "Inventory Service" as INV
participant "Message Broker"    as Q
participant "Search Service"    as SRCH

INV -> Q : StockChanged v=5 (in_stock=true)
INV -> Q : StockChanged v=6 (in_stock=false)

note over Q : v=6 delivered first\n(normal fast path)
Q -> SRCH : StockChanged v=6\n→ projection: in_stock=false ✅

note over Q : v=5 requeued due to transient error\ndelivered late
Q -> SRCH : StockChanged v=5\n→ overwrites with in_stock=true ❌

note over SRCH
  Shows in_stock=true
  Actual stock level: 0
end note
@enduml
```

---

## ACT 3 — Reads

---

### P-09 — Shared Read Replica "Just for Reads"

**Tradr did:**
Set up a read replica aggregating all services' tables. Finance ran cross-service JOINs against it for monthly reporting.

**Why it felt right:**
"It's read-only — no write conflicts, no production traffic risk. It's just reports."

**What broke:**
The Shipping team renamed `carrier_code` → `carrier_id`. Finance's JOIN on the replica broke immediately. Finance filed a support ticket against Shipping. Shipping had no idea Finance was reading their table.

Then Inventory migrated from Postgres to DynamoDB. The replica could no longer include Inventory data. Finance rewrote months of reports.

**The pattern:**
A shared read replica is a shared database with delayed writes. Schema coupling is identical — just less visible until something changes.

**Fix:** Finance builds a Finance-specific projection from events. The dependency shifts from Shipping's table structure to Shipping's event contract. A column rename in Shipping emits a new event version; Finance updates its consumer independently.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

node "Catalog Service"  as CAT
node "Shipping Service" as SHIP
database "Catalog DB"   as CAT_DB
database "Shipping DB"  as SHIP_DB
database "Shared Read\nReplica ⚠️" as REPLICA
node "Finance Reports"  as FIN

CAT  --> CAT_DB
SHIP --> SHIP_DB
CAT_DB  ..> REPLICA : replicates
SHIP_DB ..> REPLICA : replicates
FIN --> REPLICA : SELECT o.*, s.carrier_code\nJOIN ...

note right of SHIP
  Shipping renames carrier_code → carrier_id.
  Finance query breaks immediately.
  Shipping didn't know Finance was reading their table.
end note
@enduml
```

---

### P-10 — No Projection Rebuild Strategy

**Tradr did:**
The Search service built a CQRS-lite projection from `CatalogProductUpdated` events. It worked well in production for months.

**Why it felt right:**
Events keep the projection fresh. There is no need to think about the past — the projection always converges from current events.

**What broke:**
Six months in, the Search team added a `rating_score` column. Backfilling historical products required replaying all past `CatalogProductUpdated` events. The broker retained only 7 days. The history was gone.

Rebuilding from scratch meant calling the Catalog API for 4 million products — an unplanned bulk export that neither team had designed. Catalog went into brownout. The Search index was stale for 6 hours.

**The pattern:**
A projection that cannot be rebuilt is an operational liability. Event log retention is finite. Schema changes, disaster recovery, and new consumers all require replay capability.

**Fix:** Design for rebuild from day one: retain events beyond broker defaults (cold storage), the owning service exposes a paginated bootstrap API, and the consumer has a controlled, rate-limited replay mode.

```plantuml
@startuml
skinparam defaultFontName Arial
skinparam BackgroundColor transparent

participant "Catalog Service"     as CAT
participant "Message Broker"      as Q
participant "Search Service"      as SRCH
database    "product_search_index" as IDX

note over Q : Broker retains events\nfor 7 days only

CAT  -> Q    : CatalogProductUpdated (day 1 ... day N)
Q    -> SRCH : events delivered ✅
SRCH -> IDX  : projection updated ✅

note over SRCH : 6 months later:\nnew column added — rebuild needed

SRCH -> Q : request event replay from day 1
Q -[#red]->x SRCH : ❌ Events older than 7 days are gone

SRCH -> CAT : GET /products?page=1 (bulk export, unplanned)
note over CAT : 4M products, no rate limit designed\n→ Catalog brownout ❌
@enduml
```
