# Exercise 2 (30–35 min): Design an Eventually Consistent Order Workflow - Attendee Worksheet

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

