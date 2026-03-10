# Exercise 2 (30–35 min): Design an Eventually Consistent Order Workflow - Model Answer

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
