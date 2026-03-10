# Exercise 1 (30–35 min): Design a Retry-Safe “Create Order” API - Model Answer

## Model Answer (reference solution)

### 1) API contract

**Request**

* `POST /orders`
* Header: `Idempotency-Key: <uuid>`
* Body includes: `user_id`, `cart_id`, `amount`, `currency`, shipping details

**Responses**

* First success: `201 Created` with `{ order_id, status, created_at }`
* Replay after success: `200 OK` (or `201`—pick one) returning the **same `order_id`**
* Replay while processing:

  * Option A (sync create): block briefly then return final
  * Option B (async-friendly): `202 Accepted` with `{ order_id, status_url }`

Key property: **same key → same order_id** (replay is boring).

---

### 2) Storage design

Add an **idempotency table** (or “request journal”) plus your orders table.

**Table: `idempotency_records`**

* `user_id` (or tenant/customer)
* `endpoint` (e.g., `POST:/orders`)
* `idempotency_key`
* `request_hash` (hash of canonical request payload)
* `status` (`IN_PROGRESS`, `SUCCEEDED`, `FAILED`)
* `order_id` (reference) *and/or* `stored_response` (optional)
* `created_at`, `updated_at`, `expires_at`

**Uniqueness constraint**

* Unique index on `(user_id, endpoint, idempotency_key)`

**Why store request_hash**

* If same key is reused with a different body, return `409 Conflict` (or `422`) because the key must represent one logical action.

**TTL**

* Typical: 24 hours to 7 days depending on client retry behavior and business risk.
* Rationale: don’t keep forever (storage), but don’t keep too short (late retries happen).

---

### 3) Concurrency handling

**Problem:** two requests arrive concurrently with same key.

**Solution**

* Use the unique constraint + transactional insert:

  * Attempt to insert a new `idempotency_record` with status `IN_PROGRESS`
  * If insert succeeds: you own the work; create the order; update record to `SUCCEEDED`
  * If insert fails (already exists):

    * If `SUCCEEDED`: return stored `order_id` / response
    * If `IN_PROGRESS`: return `202` with status URL (or wait/poll)
    * If `FAILED`: return consistent failure semantics (see below)

This ensures only one creates side effects.

---

### 4) Crash safety / partial completion

Hard case: crash between side effects.

**Safer pattern**

* Record `IN_PROGRESS` before doing side effects.
* Prefer to make the “side effect” idempotent too:

  * e.g., when calling payment provider, pass the same idempotency key downstream.
* Update `idempotency_records` to `SUCCEEDED` only after you have a durable order_id (and payment result if applicable).
* On restart, if you see `IN_PROGRESS` older than a threshold:

  * reconcile by checking downstream (payment/order state)
  * then mark `SUCCEEDED` or `FAILED` deterministically

This matches the whitepaper principle: safe retry relies on idempotency; *controlled retry* is separate. 

---

### 5) Handling replay with different body

If same key, different payload:

* Return `409 Conflict` with message like:

  * “Idempotency-Key reuse with different request parameters”
* Include the original `order_id` if already succeeded (optional), but do not create a new one.

---

## Quick Share-out Questions (5 min)

* “What did you choose to return for IN_PROGRESS?”
* “What scope did you use for the uniqueness constraint?”
* “Did you store full response or only order_id? Why?”
* “How long is your TTL and what risk drove that number?”

---

If you want, I can turn this into a **one-page PDF worksheet** + a **one-page facilitator cheat sheet** (same content, formatted for printing).
