# Session 1 (2h): Idempotency & Eventual Consistency — Safe Retries and Async Systems (101)

## Outlins

### Goals

* Explain why “retry = danger” without idempotency.
* Design APIs/workflows that tolerate timeouts, retries, duplicates, and reordering.
* Understand what eventual consistency *is* (and isn’t), and how to build with it.

### Agenda (120 min)

**0–10: Hook + mental model**

* The “timeout → retry → double charge” story
* At-least-once delivery and why it’s common

**10–35: Idempotency fundamentals**

* Definition: same request applied multiple times ⇒ same result
* Not the same as “no side effects”
* Common misconception: “exactly once” guarantees
* Techniques:

  * Idempotency keys (client-provided)
  * Request IDs + server-side dedupe table
  * Natural idempotency (PUT with stable resource ID)

**35–55: Exercise 1 (design)**

* “Create Order” API: how to make it safe under retry
* Decide:

  * where idempotency key lives
  * what response to return on replay
  * TTL / storage strategy for dedupe records

**55–75: Eventual consistency 101**

* Why it happens: async replication, queues, sagas, caching
* What you can expect:

  * stale reads, out-of-order events, duplicates
* Patterns:

  * read-your-writes (when needed)
  * status endpoints / polling
  * 202 Accepted + async processing
  * versioning (ETags, revision numbers)

**75–95: Exercise 2 (scenario)**

* Workflow: order created → payment authorized → inventory reserved → shipping scheduled (async)
* Questions:

  * what states exist?
  * what does user see immediately?
  * how do you handle duplicate events?
  * what happens if step 3 fails after step 2 succeeds?

**95–105: Exercise 2+ (Saga compensation)**

* Travel reservation workflow: flight -> hotel -> payment
* Decide:

  * compensation order and idempotency rules
  * final user-visible failure state/message
  * when manual intervention is needed

**105–115: “Rules of thumb” checklist**

* When to retry vs when not to
* Where dedupe belongs (API gateway vs service vs consumer)
* Idempotency key pitfalls (scope, collisions, TTL)

**115–120: Wrap + 3 takeaways**

* Retry-safe design isn’t optional in distributed systems
* Eventual consistency is normal; design UX and APIs accordingly
* Prefer making operations idempotent over trying to prevent retries

### Handout (1-pager you can share)

* Idempotency checklist (API + consumer)
* Sample API response semantics on replay
* State machine example for async workflows

---

## Session 1 Slide Outline (Lecture Part)

### Slide - Cover
**Speaker notes:**
* "Welcome to Session 1 of Microservices 101."
* "Today is about correctness under retries and asynchronous processing: idempotency plus eventual consistency."
* "IDEMPOTENCY = Doing something many times has the same effect as doing it just once."
* "Eventual Consistency = Everything stays in sync, just not immediately, it takes a little time."

---

### Slide: Introduction of This Series

**Slide Title:** Microservices 101 series
**Slide contents**
* 101-01 Idempotency & Eventual Consistency - Safe Retries and Async Systems –
* 101-02 Resilience & Observability - Design for Failure and Debug Fast -

**Speaker notes:**

* "This series has two connected parts."
* "Session 1 focuses on correctness: how to avoid duplicates and design around eventual consistency."
* "Session 2 focuses on stability and diagnosis: containing failures and finding root causes quickly."
* "The sequence is intentional: resilience controls are most effective after correctness foundations are in place."

---

### Slide: Agenda

**Slide Title:** Agenda
**Slide contents**

* Introduction
* Idempotency 101
* Exercise 1 (design)
* Eventual Consistency 101
* Exercise 2 (scenario)
* Exercise 2+ (Saga)
* Rules of thumb + wrap up

**Speaker notes**

* “Two halves today: safe retries (idempotency) and async reality (eventual consistency).”
* “Retry policy tuning is Session 2; today is about making retries safe by design.”
* “Transition: let’s set the baseline for what you should be able to do by the end.”

---

### Slide: Idempotency & Eventual Consistency 101

**Slide Title:** Idempotency & Eventual Consistency 101
**Key message:** Safe retries + async reality are foundational hygiene.

**Slide contents**
**What you’ll learn**

* Why retries create duplicates
* How to design idempotent APIs and consumers
* What eventual consistency means in practice

**What you’ll be able to do**

* Make an operation safe under retry
* Explain “stale reads” without panic

**Speaker notes**

* “This is practical production hygiene, not academic distributed systems theory.”
* “In real systems, retries are unavoidable: clients retry, gateways retry, and queues redeliver. So duplicates are a design input, not an edge case.”
* “Idempotency gives us correctness under retry: same intent should converge to the same business outcome.”
* “Eventual consistency gives us the async reality: state may be temporarily out of sync, but should converge predictably.”
* “By the end of this session, you should be able to design retry-safe operations and explain temporary inconsistency without panic.”
* “Transition: let’s ground this with a simple incident story where one timeout caused duplicate side effects.”

---

### Slide: The Incident That Teaches Everything

**Slide Title:** The Incident That Teaches Everything
**Key message:** Most duplicate bugs start with a timeout.

**Slide contents**

* User clicks “Pay” → request times out → client retries
* Backend processes both → double charge / double order
* Root cause: **at-least-once**, not **exactly-once**

**Speaker notes**

* “Timeout means uncertainty, not 'the server definitely failed.'”
* “Given uncertainty, retry is rational, so duplicates are expected.”
* “Transition: here are the common places those duplicates come from.”

---

### Slide: In Distributed Systems, Duplicates Are Normal

**Slide Title:** Duplicates Are Normal
**Key message:** You don’t get to ‘turn off’ retries.

**Slide contents**
**Why retries happen**

* Client timeout
* Load balancer / gateway retry
* Queue redelivery
* Service restart mid-processing

**Therefore**

* “I sent it once” ≠ “It was processed once”
* Design assuming **at-least-once delivery**

**Speaker notes**

* “Systems prefer at-least-once delivery over losing work, so duplicates are a built-in tradeoff.”
* “Retries are part of failure dynamics, especially under stress.”
* “Transition: the core fix on the API path is the idempotency key pattern.”

---

### Slide: The Core Pattern — Idempotency Key

**Slide Title:** The Idempotency Key Pattern
**Key message:** One human intent → one durable outcome, even across retries.

**Slide contents**
**Client sends**

* `Idempotency-Key: <uuid>`
* Reuses the same key on retry

**Server guarantees**

* First request creates side effects
* Repeats return the **same outcome** (or compatible response)

**Server needs**

* Key storage + stored outcome (or reference)
* TTL / cleanup policy

**Speaker notes**

* “Idempotency keys make 'maybe processed' safe to retry.”
* “Mental model: check key, replay if seen, execute and record if new.”
* “Mindset shift: in a single DB, people rely on strong transactions or even 2PC-style coordination; in distributed systems, we design for retries, partial failure, and idempotent outcomes instead.”
* “Transition: next question is what response we should return on replay.”

---

### Slide: API Semantics — What Should Replay Return?

**Slide Title:** What Do We Return on Replay?
**Key message:** Replay should be boring.

**Slide contents**
**Recommended**

* Return original success (same `order_id`)
* Or return “still processing” with a stable status link
* Or return “something wrong, your request has been canceled”

**Avoid**

* Creating a new resource on replay
* Returning a different outcome for the same key

**Speaker notes**

* “Rule of thumb: replay should be boring and deterministic. Same idempotency key, same logical result.”
* “Preferred option one: return the original success response, including the same `order_id`.”
* “Preferred option two: if work is still in progress, return a stable in-progress response with the same `order_id` or `status_url`.”
* “If the request reaches a terminal failure state, return a consistent canceled/failed response for replays instead of retrying side effects again.”
* “What we must avoid: creating a new resource or returning a different outcome for the same key, because that breaks retry safety.”
* “Transition: this same determinism principle also applies on the consumer side.”

---

### Slide: Idempotency Mindset Shift

**Slide Title:** From ACID Habits to Distributed Reality
**Key message:** In distributed systems, assume uncertainty and make outcomes replay-safe.

**Slide contents**
**ACID (single database transaction)**

* Atomicity: all or nothing
* Consistency: rules/invariants stay valid
* Isolation: concurrent transactions do not interfere
* Durability: committed data is persisted

**Traditional instinct**

* “Wrap everything in one consistent transaction”
* Depend on strict commit coordination

**Distributed reality**

* Timeouts create uncertainty
* Cross-service global transactions are impractical at scale
* Correctness comes from idempotency + explicit state

**Speaker notes**

* “Many builders start with strong transaction assumptions from monolith/database work.”
* “In distributed apps, that assumption breaks under latency, retries, and partial failure.”
* “So the mindset changes: do local commits, expose state, and make repeated requests converge to the same outcome.”
* “Why this shift matters: in distributed systems, you cannot guarantee the real-time state of services you do not control.”
* “Transition: now let’s pause for Exercise 1 before adding async complexity.”

---

### Slide: Exercise 1 (Divider)

**Slide Title:** Exercise 1
**Slide contents:** (divider)

See session01-exercise1.md

---

### Slide: Now Add Asynchrony…

**Slide Title:** Now Add Asynchrony
**Key message:** Async improves resilience/capacity but introduces temporary inconsistency.

**Slide contents**
**Why async exists**

* Performance (don’t block request)
* Reliability (queue + retries)
* Long-running tasks

**Consequence**

* Different parts of the system update at different times

**Speaker notes**

* “Async improves throughput and resilience by avoiding long blocking calls.”
* “Tradeoff: different components update at different times.”
* “Transition: that leads directly to eventual consistency.”

---

### Slide: What Eventual Consistency Means

**Slide Title:** Eventual Consistency
**Key message:** If updates stop, the system converges—eventually.

**Slide contents**
**What you might observe**

* Stale reads
* “I wrote it but can’t see it yet”
* Out-of-order events

**What it does NOT mean**

* “Data is random”
* “Anything can happen”

**Speaker notes**

* “Eventual consistency is about timing, not random correctness.”
* “In async workflows, temporary mismatch is normal and expected.”
* “Transition: let’s name the three symptoms you should design for.”

---

### Slide: Common Consistency Symptoms

**Slide Title:** The Big Three Symptoms
**Key message:** Design for stale, out-of-order, duplicate.

**Slide contents**

* **Stale:** read doesn’t reflect latest write (yet)
* **Reordered:** event B arrives before event A
* **Duplicate:** same event delivered twice

**Speaker notes**

* “Assume stale reads, out-of-order events, and duplicates by default.”
* “If your design survives these three, it usually survives production.”
* “Transition: here are API patterns that make this manageable for clients.”

---

### Slide: API Patterns for Async Systems

**Slide Title:** API Patterns for Async + Eventually Consistent Systems
**Key message:** Make “in progress” a first-class state.

**Slide contents**

* `202 Accepted` for async initiation
* Status endpoint (`GET /.../status`)
* State machine: Pending → Processing → Succeeded/Failed
* Client UX: polling or callback/webhook

**Concrete example (Travel Reservation)**

* `POST /travel-reservations` with itinerary + payment method
* Response: `202 Accepted` + `reservation_id=TRV-123` + `status_url=/travel-reservations/TRV-123/status`
* Status transitions:
  `PENDING` -> `BOOKING_FLIGHT` -> `BOOKING_HOTEL` -> `PAYMENT_AUTHORIZING` -> `CONFIRMED`
  (or `FAILED`)
* Client behavior:
  poll `GET /travel-reservations/TRV-123/status` every 2-3s
* Example status payload while in progress:
  `{ "reservation_id": "TRV-123", "status": "BOOKING_HOTEL", "flight": "RESERVED", "hotel": "PENDING", "payment": "NOT_STARTED" }`

**Speaker notes**

* “`202 Accepted` means accepted now, finished later.”
* “A status endpoint gives clients a stable contract during async processing.”
* “In the travel flow, `TRV-123` is the anchor for retries and progress checks.”
* “Transition: now let’s handle partial failure with Saga compensation.”

---

### Slide: Saga Pattern (Compensation)

**Slide Title:** Saga Pattern for Distributed Workflows
**Key message:** When a later step fails, compensate earlier steps to restore business consistency.

**Slide contents**
**When to use Saga**

* Multi-step workflow across services
* No global transaction across all steps
* Partial failure is possible and expected

**Compensation model**

* Step fails -> trigger compensating action for completed steps
* Example: payment authorized but inventory failed -> void/refund payment
* Keep forward + compensating actions idempotent

**Concrete example (Travel Reservation Saga)**

* Forward steps:
  1. Reserve flight (`flight_reservation_id=FL-777`)
  2. Reserve hotel (`hotel_reservation_id=HT-555`)
  3. Authorize payment (`payment_auth_id=PM-999`)
* Failure scenario:
  payment authorization fails (card declined)
* Compensation steps:
  1. Cancel hotel reservation (`HT-555`)
  2. Cancel flight reservation (`FL-777`)
  3. Mark travel reservation as `FAILED_PAYMENT`
* Idempotency requirement:
  repeated `CancelHotel(HT-555)` or `CancelFlight(FL-777)` must be safe and return same final state

**Speaker notes**

* “Saga means we recover from partial failure and converge to a valid business state.”
* “Each step needs a forward action and a compensation action.”
* “Compensation calls must also be idempotent because retries happen there too.”
* “Transition: let’s combine idempotency and async patterns into one recipe.”

---

### Slide: Exercise 2 (Divider)

**Slide Title:** Exercise 2
**Slide contents:** (divider)

See session01-exercise2.md

---

### Slide: Exercise 2+ (Divider)

**Slide Title:** Exercise 3
**Slide contents:** (divider)

See session01-exercise3.md

---

### Slide: The Combined Recipe

**Slide Title:** The Combined Recipe
**Key message:** Retries are safe only when operations are idempotent; async requires explicit state.

**Slide contents**
**When you add retries**

* Enforce idempotency at side-effect boundaries

**When you add async**

* Expose state + make progress observable

**Starter kit**

* Request ID / idempotency key
* Dedupe storage / unique constraint
* State machine + status endpoint

**Speaker notes**

* “Core line: **Retries are safe only when operations are idempotent.**”
* “Add async state visibility, and you get a solid baseline design.”
* “Transition: before we wrap, here are common failure traps.”

---

### Slide: Failure Modes to Watch For

**Slide Title:** Pitfalls and Anti-patterns
**Key message:** Most bugs come from missing one detail.

**Slide contents**

* Key not scoped to user + operation
* TTL too short → late retry creates duplicates
* Returning different outcomes for same key
* Partial completion not recorded safely
* Assuming event ordering

**Speaker notes**

* “Scope keys correctly to avoid collisions and cross-tenant leakage.”
* “Set TTL for real retry timing, including late mobile/background retries.”
* “Design for partial completion and explicit ordering where needed.”
* “Transition: final slide connects this to resilience controls.”

---

### Slide: Bridge to Resilience (Alignment with Whitepaper)

**Slide Title:** Where This Connects to Resilience
**Key message:** Idempotency makes retries safe; resilience makes retries controlled.

**Slide contents**

* Safe retry → **Idempotency** (Session 1)
* Controlled retry → attempts + backoff + jitter + timeouts (Session 2)
* Uncontrolled retry can accelerate collapse (retry amplification)

**Speaker notes**

* “This connects directly to controlled retry: bounded attempts, backoff, jitter, and idempotency.”
* “Without control, retries amplify load and can trigger cascades.”
* “Retry policy is a resilience control, not a convenience feature.”

---

### Handout (1-pager you can share)

* Idempotency checklist (API + consumer)
* Sample API response semantics on replay
* State machine example for async workflows
