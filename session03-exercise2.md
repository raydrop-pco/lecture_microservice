# Session 3 — Exercise 2: Change Propagation Pipeline (15 min)

## Setup (what attendees get)

Design a reliable event-driven data-copy pipeline using the patterns from this session: **outbox**, **inbox**, and **version check**.

**Scenario**

* The **Catalog** service is the system of record for product names and descriptions.
* Two downstream services maintain their own denormalized read models:
  * **Search** — indexes product name + description for full-text search
  * **Pricing** — displays the product name alongside pricing info
* When Catalog updates a product name, both consumers must reflect the change **reliably** — without shared DB access.
* Events are delivered **at-least-once** (duplicates possible).
* Events may arrive **out of order** (e.g., during retries or redelivery).

**Assumptions**

* Each service owns its own database (DB-per-service).
* A message broker sits between services (e.g., Kafka, RabbitMQ).
* Eventual consistency is acceptable — consumers may lag, but must converge to the correct state.
* You can use any patterns covered in this session (outbox, inbox, version check, CDC).

---

# Attendee Worksheet (printable)

### Part A — Owner-Side Write + Outbox Transaction (5 min)

The Catalog service updates a product name. Design the **single local transaction** that:

1. Updates the product row
2. Inserts an event into the outbox table

Fill in the blanks:

```sql
BEGIN;

UPDATE __________ SET __________ = __________ , version = version + 1
  WHERE product_id = __________ ;

INSERT INTO __________ (id, aggregate_type, aggregate_id, event_type, payload, created_at)
VALUES (
  __________ ,          -- unique event ID
  __________ ,          -- aggregate type
  __________ ,          -- aggregate ID (product_id)
  __________ ,          -- event type name
  __________ ,          -- JSON payload
  now()
);

COMMIT;
```

Questions:

* Why must both statements be in the **same transaction**?
  * Answer: ___________________________________
* What happens if the application crashes **after UPDATE but before INSERT**?
  * Answer: ___________________________________

---

### Part B — Define the Event Contract (3 min)

Design the event that the relay publishes to the broker.

Fill in the event schema:

| Field | Value / Type | Purpose |
|-------|-------------|---------|
| `event_id` | `UUID` | ________________________ |
| `event_type` | `________________` | ________________________ |
| `product_id` | `UUID` | ________________________ |
| `version` | `integer` | ________________________ |
| `name` | `string` | ________________________ |
| `description` | `string` | ________________________ |
| `occurred_at` | `timestamp` | ________________________ |

Questions:

* Is this a **full-state** event or a **delta** event? Why does it matter?
  * Answer: ___________________________________
* What should the broker use as the **partition key**? Why?
  * Answer: ___________________________________

---

### Part C — Consumer Update Logic (5 min)

Design the consumer-side logic for the **Search** service (Pricing follows the same pattern).

For each incoming event, the consumer must:

1. **Check for duplicates** (inbox pattern)
2. **Check for stale events** (version check pattern)
3. **Update the projection**

Fill in the pseudocode:

```
on receive ProductNameChanged(event):

  -- Step 1: Dedup
  result = INSERT INTO __________ (message_id, processed_at)
           VALUES (event.__________, now())
           ON CONFLICT DO __________

  if result == __________ :
      return  -- already processed

  -- Step 2: Version check
  current_version = SELECT __________ FROM __________
                    WHERE product_id = event.product_id

  if event.version __________ current_version:
      return  -- stale event, skip

  -- Step 3: Update projection
  UPSERT INTO __________ (product_id, name, description, last_applied_version)
  VALUES (event.product_id, event.name, event.description, event.__________)
```

Questions:

* What happens if a duplicate event arrives?
  * Answer: ___________________________________
* What happens if an older version arrives after a newer one?
  * Answer: ___________________________________
* Should Steps 1–3 be in a **single transaction**? Why?
  * Answer: ___________________________________

---

### Part D — Failure Drill (2 min)

Walk through this failure scenario:

> The relay publishes event `version=5` to the broker. The broker delivers it to Search. Search processes it and updates its view. Then the broker **redelivers the same event** (at-least-once guarantee).

Answer:

1. What does the consumer do on the second delivery?
   * ___________________________________
2. What is the final state of the Search projection?
   * ___________________________________
3. Is data consistency preserved?
   * ___________________________________

---

### Bonus (if time)

Draw a **sequence diagram** (on paper or whiteboard) showing the full pipeline:

```
Catalog write → outbox insert → relay → broker → Search consumer → inbox check → version check → projection update
```

Include one duplicate delivery and one out-of-order delivery in your diagram.

---

---

# Session 3 — Exercise 2 Facilitation Guide

## Objective

**Design and walk through a complete change-propagation pipeline** that handles the two failure modes covered in this session: duplicate delivery and out-of-order events.

Key learning: The outbox guarantees "no event lost" on the producer side. The inbox + version check guarantees "duplicates and stale events are harmless" on the consumer side. Together, they form a reliable data-copy mechanism under eventual consistency.

---

## Timing (15 min)

* **1 min:** Intro + scenario recap
* **5 min:** Part A (outbox transaction)
* **3 min:** Part B (event contract)
* **5 min:** Part C (consumer logic)
* **1 min:** Part D (failure drill)

---

## Facilitation Strategy

### Part A — Owner-Side Write + Outbox Transaction (5 min)

**Opening framing:**

> "You're the Catalog service. A user just renamed a product. You need to persist the change AND guarantee an event gets published. How do you do both atomically?"

**Key questions to ask groups:**

* "What happens if you UPDATE the product but the outbox INSERT fails? Is the event lost?"
  * Answer: If in the same transaction, both roll back — no inconsistency.
* "What happens if you publish to the broker directly instead of using an outbox?"
  * Answer: That's the dual-write problem from Slide 27. The DB write can succeed but the broker publish can fail, or vice versa.
* "Does the relay need to be transactional?"
  * Answer: No — relay is fire-and-forget. If it crashes, it retries from where it left off. The consumer handles duplicates.

---

### Part B — Define the Event Contract (3 min)

**Key questions:**

* "Is this event full-state or delta?"
  * Full-state: carries the current name + description. Consumer can overwrite without history.
  * Delta: "name changed from X to Y." Requires ordering to apply correctly. Much harder.
  * **For 101 level: full-state is the right choice.**
* "Why partition by product_id?"
  * All events for the same product go to the same partition → broker preserves per-entity order.
* "Why include version?"
  * Enables the consumer to detect and skip stale events (version check pattern from Slide 30).

---

### Part C — Consumer Update Logic (5 min)

**Opening framing:**

> "You're the Search service. An event just arrived. You don't know if it's new, a duplicate, or outdated. Design your logic to handle all three cases."

**Key questions:**

* "What if you skip the inbox check and just do the version check?"
  * You'd process the same event twice — the version check would pass both times (same version). The upsert would be idempotent only if your SQL happens to be an upsert. Inbox is the explicit safety net.
* "What if you skip the version check and just do the inbox check?"
  * Duplicates are safe, but a late-arriving older event would overwrite newer data. The inbox only tracks message_id, not ordering.
* "Should all three steps be in one transaction?"
  * Yes — otherwise a crash between inbox insert and projection update leaves the event marked as "processed" but the view is stale. On retry, the inbox says "already done" and the update is lost.

---

### Part D — Failure Drill (1 min)

**Walk through as a group:**

1. Event `version=5` arrives → inbox check passes (new) → version check passes (5 > current) → projection updated to version 5
2. Same event redelivered → inbox check: `message_id` already exists → **skip** → projection unchanged
3. Final state: version 5, correct data, no corruption

**Quick follow-up:**

> "Now imagine version 4 arrives after version 5 — what happens?"

Answer: Inbox check passes (new message_id), but version check catches it (4 < 5) → skip. Projection stays at version 5.

---

## Common Pitfalls & How to Address Them

| Pitfall | What attendees often do | How to redirect |
|---------|--------------------------|-----------------|
| **Publish directly to broker in the transaction** | `BEGIN; UPDATE ...; publish(); COMMIT;` | "What if publish() succeeds but COMMIT fails? Or publish() fails but COMMIT already happened? That's the dual-write problem." |
| **Skip the outbox — "just retry on failure"** | Assume application-level retry is sufficient | "What if the app crashes before retry? The event is lost. Outbox guarantees it's persisted." |
| **Inbox only, no version check** | Assume dedup is enough | "Dedup prevents processing the same event twice. It doesn't prevent an older event from overwriting a newer one." |
| **Version check only, no inbox** | Assume version comparison handles everything | "If the same version arrives twice and both pass the version check, the projection runs twice. Usually harmless for upserts, but inbox makes the contract explicit." |
| **Separate transactions for inbox + projection** | Insert inbox, then update projection in a second transaction | "If you crash between them, the inbox says 'done' but the projection is wrong. Single transaction or nothing." |
| **Delta events instead of full-state** | "Event: name changed from A to B" | "Delta events require strict ordering. Full-state events are idempotent — the consumer just overwrites with the latest." |

---

## Debrief Questions (share-out)

Ask the room / breakout groups:

1. **"What does the outbox guarantee that a direct broker publish doesn't?"**
   * Atomic persistence — the event is committed in the same transaction as the data change.

2. **"What are the two failure modes the consumer must handle?"**
   * Duplicate delivery (→ inbox) and out-of-order delivery (→ version check).

3. **"Why do we use full-state events instead of deltas?"**
   * Full-state is idempotent — apply it any number of times, get the same result. Deltas require strict ordering.

4. **"If the broker already guarantees per-partition ordering, do you still need the version check?"**
   * In theory, no. In practice, yes — retries, consumer restarts, and rebalancing can break ordering. The version check is a safety net.

5. **"How does this pipeline compare to a shared database?"**
   * The shared DB gives immediate consistency but couples all services. This pipeline gives eventual consistency with full ownership and independent deployability.

---

# Model Answers

### Part A — Outbox Transaction

```sql
BEGIN;

UPDATE products SET name = 'Widget Pro', version = version + 1
  WHERE product_id = 'prod-42';

INSERT INTO outbox (id, aggregate_type, aggregate_id, event_type, payload, created_at)
VALUES (
  'evt-uuid-001',
  'Product',
  'prod-42',
  'ProductNameChanged',
  '{"product_id": "prod-42", "name": "Widget Pro", "description": "Updated widget", "version": 6}',
  now()
);

COMMIT;
```

* Both in the same transaction → if either fails, both roll back. No orphaned events, no lost updates.
* If the app crashes after UPDATE but before COMMIT → transaction rolls back. No partial state.

---

### Part B — Event Contract

| Field | Value / Type | Purpose |
|-------|-------------|---------|
| `event_id` | `UUID` (`evt-uuid-001`) | Unique ID for deduplication (inbox key) |
| `event_type` | `ProductNameChanged` | Consumer uses this to route processing logic |
| `product_id` | `UUID` (`prod-42`) | Identifies which product changed; broker partition key |
| `version` | `integer` (`6`) | Monotonic counter for version check (stale detection) |
| `name` | `string` (`"Widget Pro"`) | Current full state — not a delta |
| `description` | `string` (`"Updated widget"`) | Current full state |
| `occurred_at` | `timestamp` | Informational — not used for ordering (version is) |

* **Full-state event:** consumer can overwrite without needing prior state or history.
* **Partition key = `product_id`:** all events for the same product land on the same partition → broker preserves per-entity order.

---

### Part C — Consumer Logic (Search Service)

```
on receive ProductNameChanged(event):

  -- Step 1: Dedup
  result = INSERT INTO inbox (message_id, processed_at)
           VALUES (event.event_id, now())
           ON CONFLICT DO NOTHING

  if result == already_exists:
      return  -- duplicate, skip

  -- Step 2: Version check
  current_version = SELECT last_applied_version FROM search_products
                    WHERE product_id = event.product_id

  if event.version <= current_version:
      return  -- stale event, skip

  -- Step 3: Update projection
  UPSERT INTO search_products (product_id, name, description, last_applied_version)
  VALUES (event.product_id, event.name, event.description, event.version)
```

* Duplicate → inbox catches it (message_id already exists) → skip
* Stale → version check catches it (event.version ≤ current) → skip
* All three steps in **one transaction** — crash-safe

---

### Part D — Failure Drill

1. First delivery of `version=5`: inbox insert succeeds → version check passes (5 > current) → projection updated → ✓
2. Redelivery of same event: inbox insert fails (duplicate key) → skip → projection unchanged → ✓
3. Final state: `version=5`, correct data, no corruption → ✓

**Bonus — out-of-order:** `version=4` arrives after `version=5`:
* Inbox: passes (new message_id)
* Version check: 4 ≤ 5 → skip
* Projection stays at version 5 → ✓
