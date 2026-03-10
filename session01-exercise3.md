## Exercise 2+ (10 min): Saga Compensation for E-commerce Orders

### Goal

Design a compensation strategy for a partially failed async order workflow.

### Scenario

An order runs through async steps:

1. `PaymentAuthorized`
2. `InventoryReserved`
3. `ShipmentScheduled`

Reality constraints:

* Messages are delivered at least once (duplicates can happen).
* Events can be delayed or out of order.
* Compensation commands can also be retried.
* Inventory reservation may be partial in some failures.

---

## Attendee Worksheet (printable)

### Part A - Failure Path Design (4 min)

Failure case: payment is authorized, then inventory reservation fails.

Decide:

* Compensation order
* Final order status
* What the customer should see

Fill in:

* Compensation action 1: `____________________`
* Compensation action 2 (if needed): `____________________`
* Final status: `____________________`
* User message: `____________________`

### Part B - Idempotency Rules for Compensation (3 min)

Define idempotency guarantees for:

* `VoidPayment(payment_auth_id)`
* `ReleaseInventory(order_id, sku)`

Rule format:

* "If command is repeated, system returns `________` and state remains `________`."

### Part C - Edge Cases (3 min)

For each case, define behavior:

1. `VoidPayment` times out after inventory is already marked failed.
2. Duplicate `InventoryFailed` event arrives twice.
3. `VoidPayment` returns `already_voided`.

For each, fill:

* System status: `____________________`
* Retry/next action: `____________________`
* Manual intervention needed? `Yes / No` (why): `____________________`

---

## Facilitation Guide

### Timing

* 1 min: restate scenario and constraints
* 4 min: Part A
* 3 min: Part B
* 2 min: Part C + quick share

### Prompts to Ask

* "What exact state means compensation is complete?"
* "How do you avoid double-voiding payment on duplicate events?"
* "When do you escalate to manual intervention instead of retrying forever?"

### Common Mistakes

* No explicit terminal compensation state
* Compensation commands not idempotent
* Duplicate `InventoryFailed` triggers duplicate side effects
* No decision rule for unresolved timeout uncertainty

---

## Model Answer

### A) Failure Path

Recommended flow:

1. Receive `InventoryFailed(order_id)`
2. Trigger `VoidPayment(payment_auth_id)`
3. If any inventory was partially reserved, trigger `ReleaseInventory(order_id, sku)`
4. Mark order `FAILED_INVENTORY_COMPENSATED`

Customer-facing message:

* "Item is out of stock. Your payment authorization was voided."

### B) Idempotency Rules

* `VoidPayment(payment_auth_id)`:
  repeated command returns success-equivalent (`VOIDED` or `ALREADY_VOIDED`) and state remains `VOIDED`.
* `ReleaseInventory(order_id, sku)`:
  repeated command returns success-equivalent (`RELEASED` or `ALREADY_RELEASED`) and state remains `RELEASED`.

### C) Edge Case Handling

1. `VoidPayment` times out:
   keep status `COMPENSATION_IN_PROGRESS`, retry with same command ID, then reconcile with payment provider before closing.
2. Duplicate `InventoryFailed` event:
   dedupe by `event_id`; if already processed, no additional side effects.
3. `already_voided` response:
   treat as successful compensation and continue to terminal failed-compensated state.

### Strong Team Output Checklist

* Compensation order is explicit and deterministic.
* Terminal state is business-safe (`FAILED_INVENTORY_COMPENSATED` or similar).
* Compensation operations are idempotent under retries.
* Duplicate failure events are harmless.
* Manual intervention is only for unresolved uncertainty after bounded retries/reconciliation.
