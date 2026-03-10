# Exercise 2+ (10 min): Saga Compensation for E-commerce Orders - Attendee Worksheet

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

