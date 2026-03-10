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

### 3) Concurrency + crash safety

**No.1 Question:** Two identical requests (same idempotency key) arrive at the same time. How do you prevent creating two orders?

**No.1 Model answer**

* Enforce a unique constraint on `(user_id, endpoint, idempotency_key)`.
* In a transaction, try to insert an `idempotency_record` with status `IN_PROGRESS`.
* If insert succeeds: this request owns processing, creates the order, then updates status to `SUCCEEDED` with `order_id`.
* If insert fails (record already exists):

  * If status is `SUCCEEDED`, return the stored `order_id` / response.
  * If status is `IN_PROGRESS`, return `202 Accepted` (or wait briefly and return final result).
* Do not run order creation logic in the duplicate request path.

Result: only one request can create side effects; concurrent duplicates become safe replays.

**No.2 Question:** The service crashes after charging payment but before returning a response. What do you need to record, and when, to avoid inconsistent outcomes?

**No.2 Model answer**

* Insert idempotency record as `IN_PROGRESS` before external side effects (like payment).
* Call downstream services (payment) with the same idempotency key so retries are also safe downstream.
* Persist durable milestones (`payment_attempt_id`, `order_id`, final payment state) before marking success.
* Mark idempotency record `SUCCEEDED` only after durable success state is written.
* On recovery/retry, if record is stale `IN_PROGRESS`, reconcile by checking payment/order state, then deterministically set `SUCCEEDED` or `FAILED` and return consistent result.

**No.3 Question:** What do you do if replay request body differs from the original (same key, different amount/cart)?

**No.3 Model answer**

* Compare incoming request hash to stored `request_hash` for that key.
* If different, reject with `409 Conflict` (or `422 Unprocessable Entity`) and do not create a new order.
* Return a clear error message, for example: `Idempotency-Key reuse with different request parameters`.

---

### 4) Bonus: Async fulfillment (Part D)

If order fulfillment is asynchronous, a clean model is:

* Return `202 Accepted` from `POST /orders` when work is queued, with `{ order_id, status: "PENDING", status_url }`.
* Add `GET /orders/{order_id}` (or `GET /jobs/{job_id}`) as the status endpoint.
* Keep one stable identifier per idempotency key:

  * Same key + same request must always return the same `order_id` (or `job_id`).
  * Replays return current status (`PENDING`, `PROCESSING`, `SUCCEEDED`, `FAILED`) for that same identifier.
* Persist mapping in `idempotency_records`:

  * key -> `order_id` / `job_id` + current state + timestamps.
* When processing completes, update final state; subsequent replays/status checks read the stored result rather than creating new work.

This keeps retries safe while supporting long-running workflows.

---

## Quick Share-out Questions (5 min)

* “What did you choose to return for IN_PROGRESS?”
* “What scope did you use for the uniqueness constraint?”
* “Did you store full response or only order_id? Why?”
* “How long is your TTL and what risk drove that number?”

---

If you want, I can turn this into a **one-page PDF worksheet** + a **one-page facilitator cheat sheet** (same content, formatted for printing).
