## Exercise 2 (30–35 min): Design an Eventually Consistent Order Workflow

---

### Goal

Design a simple distributed workflow where:

* processing is **asynchronous**
* reads may be **stale**
* events may be **duplicated** or **out of order**
  …and your API + state model still behave predictably.

---

## Attendee Worksheet (printable)

### Scenario

You run an e-commerce system. Creating an order triggers downstream steps:

1. **Payment Authorized** (Payment service)
2. **Inventory Reserved** (Inventory service)
3. **Shipment Scheduled** (Shipping service)

To keep request latency low, `POST /orders` returns quickly and the rest happens asynchronously via events.

**Reality constraints**

* Events can be delivered **at least once** (duplicates possible).
* Events can arrive **out of order**.
* Reads may be **eventually consistent** (status may lag).
* Any step might fail transiently or permanently (don’t focus on retry policy—focus on correctness and state).

---

### Part A — Define your order state machine (10 min)

1. List order states (minimum 5–7 states). Example format:

* `PENDING_PAYMENT`
* `PAYMENT_AUTHORIZED`
* `INVENTORY_RESERVED`
* …

2. Draw transitions:

* What events cause state to move forward?
* What failure states exist? (e.g., `PAYMENT_FAILED`, `CANCELLED`, `FULFILLMENT_FAILED`)

3. Define “terminal states” (no further transitions).

---

### Part B — API contract for eventual consistency (10 min)

Design **two endpoints**:

1. `POST /orders` response

* Status code: `____` (201 or 202)
* Body: include `order_id`? `status_url`? initial status?

2. `GET /orders/{order_id}` (or `/status`)

* What does it return?
* How do clients know it’s still processing vs stuck vs failed?

3. UI/Client behavior (one sentence)

* “What should the frontend show right after create?”

---

### Part C — Handle duplicates & out-of-order events (10 min)

You receive events like:

* `PaymentAuthorized(order_id, event_id, occurred_at, …)`
* `InventoryReserved(order_id, event_id, occurred_at, …)`
* `ShipmentScheduled(order_id, event_id, occurred_at, …)`

1. **Dedup strategy**

* What ID do you dedup on?
* Where do you store it?

2. **Out-of-order strategy**
   Pick one (or propose your own):

* ( ) Ignore older events using a version/sequence
* ( ) Use “state guards” (only accept valid transitions)
* ( ) Reorder by `occurred_at` (dangerous in practice)
* ( ) Other: __________

3. Define the rule:

* “If we receive ShipmentScheduled before InventoryReserved, we will ________.”

---

### Part D — Failure handling in an eventually consistent workflow (5 min)

Pick one failure:

* Inventory reservation fails after payment succeeded

Answer:

* Do you compensate (refund / release) or keep order pending?
* What status does the user see?
* What manual/ops action exists (if any)?

---

### Bonus (if time)

Define 2–3 **invariants** that must always hold. Example:

* “An order cannot be marked SHIPPED unless inventory was reserved.”

---

## Facilitation Guide (for you)

### Timing (recommended)

* 2 min intro: remind “design for stale / duplicate / out-of-order”
* 10 min Part A
* 10 min Part B
* 10 min Part C
* 5 min Part D + share-out

### Prompts to circulate with

* “What’s your single source of truth for order state?”
* “What does ‘in progress’ look like in your API?”
* “How do you prevent a duplicate event from advancing state twice?”
* “If events arrive out of order, what prevents impossible states?”

### Common mistakes to watch for

* No explicit “processing” states → client confusion
* Assuming event ordering
* Deduping in memory (fails on restart)
* Accepting any event at any time (state jumps)
* No story for partial success (payment succeeded, inventory failed)

---

## Model Answer (share after exercise)

### A) Order state machine (one solid solution)

**States**

* `CREATED` (order accepted, workflow not started or just started)
* `PAYMENT_PENDING`
* `PAYMENT_AUTHORIZED`
* `INVENTORY_PENDING`
* `INVENTORY_RESERVED`
* `SHIPPING_PENDING`
* `SHIPPING_SCHEDULED`
* Terminal success: `COMPLETED` (optional if shipping scheduled is “done”)
* Terminal failure: `FAILED_PAYMENT`, `FAILED_INVENTORY`, `FAILED_SHIPPING`
* Terminal cancel: `CANCELLED` (optional)

**Transitions**

* `CREATED → PAYMENT_PENDING` (workflow started)
* `PAYMENT_PENDING → PAYMENT_AUTHORIZED` on `PaymentAuthorized`
* `PAYMENT_PENDING → FAILED_PAYMENT` on `PaymentFailed`
* `PAYMENT_AUTHORIZED → INVENTORY_PENDING`
* `INVENTORY_PENDING → INVENTORY_RESERVED` on `InventoryReserved`
* `INVENTORY_PENDING → FAILED_INVENTORY` on `InventoryFailed`
* `INVENTORY_RESERVED → SHIPPING_PENDING`
* `SHIPPING_PENDING → SHIPPING_SCHEDULED` on `ShipmentScheduled`
* `SHIPPING_PENDING → FAILED_SHIPPING` on `ShippingFailed`

Key idea: explicit *pending* states make eventual consistency legible.

---

### B) API contract

**POST /orders**

* Prefer `202 Accepted` with:

  * `{ order_id, status: "CREATED", status_url: "/orders/{id}" }`
* Alternative: `201 Created` is fine if you create an order record synchronously; the workflow is still async.

**GET /orders/{id}**
Return:

* `status` (state machine status)
* `last_updated_at`
* optional `progress` fields (payment/inventory/shipping sub-status)
* optional `failure_reason` (sanitized)

**Client UX**

* After create: show “Order received—processing payment/inventory…” and poll status (or subscribe).

---

### C) Duplicates + out-of-order

**Dedup**

* Every event has `event_id` (globally unique).
* Maintain `processed_events(order_id, event_id)` with unique constraint.
* On event receive:

  * insert processed_events → if conflict, drop as duplicate
  * then apply state transition

**Out-of-order**
Use **state guards** + optional versioning:

* Rule: only accept transitions from allowed prior states.
* If `ShipmentScheduled` arrives before `INVENTORY_RESERVED`:

  * Do **not** advance to SHIPPING_SCHEDULED immediately.
  * Either:

    * store it as “pending shipping confirmation” and retry apply later, or
    * ignore and rely on re-delivery / subsequent event (less ideal)
      Best simple rule for entry level: **state guards + park unexpected events** (dead-letter / pending table) and alert.

---

### D) Partial failure: payment succeeded, inventory fails

Two common strategies:

1. **Compensate**: release/void payment authorization (or refund) and mark `FAILED_INVENTORY`
2. **Backorder**: keep in pending/backorder state and notify user

For this exercise, recommend compensate for simplicity:

* User sees: “Out of stock—payment voided” (or “not charged”)
* Status: `FAILED_INVENTORY` with reason code
* Ops: manual re-run or customer service override (optional)

---

### Example invariants (good answers)

* Cannot reach `INVENTORY_RESERVED` unless payment is authorized
* Cannot reach `SHIPPING_SCHEDULED` unless inventory is reserved
* Each event is applied at most once (dedup by event_id)
* Order is never both `COMPLETED` and `FAILED_*`
