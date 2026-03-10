## Exercise 1 (30–35 min): Design a Retry-Safe “Create Order” API

### Setup (what attendees get)

You’re designing an API endpoint that creates an order.

**Scenario**

* Client calls `POST /orders` to create an order.
* The network is unreliable: requests may **timeout**, and clients may **retry**.
* The backend may also crash/restart during processing.
* Requirement: **No duplicate orders** for the same user intent.

**Assumptions**

* The client can generate a UUID.
* The client *will* reuse it on retry if you tell them to.
* Backend has a relational DB available.
* Don’t solve retry policy (backoff/jitter/attempts) — focus on **correctness under retries**.

---

## Attendee Worksheet (printable)

### Part A — Define the API contract (10 min)

Fill in:

1. **Request**

* Endpoint: `POST /orders`
* Headers:

  * `Idempotency-Key: ____________________`
* Body fields (minimal):

  * `user_id`, `cart_id`, `amount`, `currency`, etc.

2. **Responses**
   Define what the API returns for:

* **First successful request**: status `____` and body `________________`
* **Replay of the same key after success**: status `____` and body `________________`
* **Replay while first request is still running**: status `____` and body `________________`

---

### Part B — Server-side storage design (10 min)

Design the minimal server-side data you need.

1. What table(s) do you add?

* Table name(s): ____________________
* Key columns: ______________________
* What do you store: full response vs reference (order_id/job_id)? Why?

2. What is the scope of idempotency?
   Pick one:

* ( ) Global (dangerous)
* ( ) Per endpoint
* ( ) Per user + endpoint (recommended)
* ( ) Other: ___________

3. TTL / retention

* How long do you keep idempotency records? ___________
* Why not forever? Why not 5 minutes?

---

### Part C — Concurrency + crash safety (10 min)

Handle these realities:

1. Two identical requests (same idempotency key) arrive **at the same time**.

* How do you prevent creating two orders?

2. The service crashes after charging payment but before returning a response.

* What do you need to record, and when, to avoid inconsistent outcomes?

3. What do you do if the replay request body differs from the original (same key, different amount/cart)?

* Behavior: ______________________

---

### Part D — Bonus (if time)

If order fulfillment is async:

* Do you return `202`?
* What status endpoint do you add?
* How does idempotency map to a stable job/order identifier?

---

## Facilitation Guide (what you say / do)

### Timing

* 2 min: Explain scenario + constraints
* 10 min: Part A
* 10 min: Part B
* 10 min: Part C
* 5 min: Share-out + compare to model

### Prompts to circulate with

Ask teams:

* “What exactly is the *same logical action* here?”
* “What do you return on replay so it’s deterministic?”
* “Where is the ‘side-effect boundary’?”
* “What prevents two concurrent creates?”

### Common mistakes to watch for

* Using idempotency key but not scoping it (user collision)
* Only deduping in memory (breaks on restart)
* No concurrency control (race creates duplicates)
* Ignoring “same key, different body”
* TTL too short without reasoning

---

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
