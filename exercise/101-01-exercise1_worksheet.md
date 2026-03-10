# Exercise 1 (30–35 min): Design a Retry-Safe “Create Order” API - Attendee Worksheet

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

