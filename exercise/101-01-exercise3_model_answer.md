# Exercise 2+ (10 min): Saga Compensation for E-commerce Orders - Model Answer

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
