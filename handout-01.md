# Session 1 Handout — Idempotency & Eventual Consistency (1-pager)

Use this as a practical checklist when designing retry-safe APIs/consumers and async workflows.  
**Focus:** correctness under retries (retry policy controls are covered in Session 2).

---

## 1) Idempotency checklist (API + consumer)

### API (request/response)
- Require an **Idempotency-Key** (client-generated UUID) for non-idempotent creates (e.g., POST that causes side effects).
- Scope dedup by **(tenant/user, endpoint, idempotency_key)** (avoid global key collisions).
- Define deterministic replay semantics: **same key ⇒ same outcome** (same resource/job id).
- Handle **IN_PROGRESS** replays:
  - return **202 Accepted** + `status_url`, or
  - block briefly then return the final result (be explicit which).
- Reject same key with different payload:
  - store a **request_hash** (canonicalized payload) and return **409 Conflict** (or **422**).
- Record idempotency state **durably** *before* side effects (don’t dedupe only in memory).
- Pick a realistic **TTL** (late retries happen) and document it.

### Consumer (queue/event handler)
- Assume **at-least-once** delivery: duplicates and redelivery will occur.
- Dedup by a stable **event_id/message_id** with a durable store + **unique constraint**.
- Apply transitions with **state guards** (only accept valid next steps).
- Make side effects idempotent:
  - prefer **upsert/unique constraints**
  - propagate request/event ids to downstream calls when possible
- Have a plan for unexpected/out-of-order events:
  - **park** (pending table), **DLQ**, or **reconcile** later.

---

## 2) Sample API response semantics on replay

| Situation | Recommended response |
|---|---|
| First request succeeds | **201 Created** (or **200 OK**) with `order_id` and status |
| Replay after success (same key) | **200 OK** (or **201**) with the **same `order_id`** (replay is “boring”) |
| Replay while processing | **202 Accepted** with **`status_url`** (or block briefly then return final) |
| Same key, different body | **409 Conflict** (or **422**); do **not** create a new order |
| Prior attempt failed (deterministic) | Return the **same failure**, or require a **new key** for a new attempt (document the rule) |

---

## 3) State machine example for async workflows

**Order workflow (example):** keep “in progress” explicit so eventual consistency is understandable to clients.

### States (happy path)
`CREATED → PAYMENT_PENDING → PAYMENT_AUTHORIZED → INVENTORY_PENDING → INVENTORY_RESERVED → SHIPPING_PENDING → SHIPPING_SCHEDULED`  
(`→ COMPLETED` optional)

### Failure terminal states (examples)
- `FAILED_PAYMENT`
- `FAILED_INVENTORY`
- `FAILED_SHIPPING`
- `CANCELLED` (optional)

### Rules of thumb
- Treat each step as an event-driven transition; apply with **state guards** (only legal next steps).
- Dedup events by **event_id** so duplicates don’t advance state twice.
- If an event arrives out of order (e.g., `ShipmentScheduled` before `InventoryReserved`), **park** it (pending/DLQ) and reconcile later rather than jumping state.
