That lineup makes a lot of sense: the two sessions share the same “distributed-systems survival skills” DNA, but they’re separable enough that people can take either one without feeling lost.

A few tweaks that would make them *even more* entry-level-friendly (and more obviously “independent”):

## 1) Idempotency & Eventual Consistency 101

This is a great “correctness under retries + async reality” session.

What works well:

* These two topics naturally pair: idempotency is *how you safely retry*, eventual consistency is *why retries / async propagation happen in the first place*.

To keep it approachable:

* Anchor everything to **one simple story**: “client times out → retries → backend processes twice → money moved twice (oops)”.
* Make sure you explicitly separate:

  * **Idempotency (behavior of an operation)** vs
  * **Deduplication (implementation technique)** vs
  * **Exactly-once (a promise people often assume incorrectly)**

Hands-on exercise idea (great for 2 hours):

* Design an “Order API”:

  * naive version → shows double-charge
  * add idempotency key / request ID → show safe retry
  * then introduce async fulfillment / event propagation → show eventual consistency and how the UI/UX or API should respond (202 Accepted, status endpoint, etc.)

## 2) Resiliency & Observability 101

Also a great pair: resiliency is *how you behave under failure*; observability is *how you know what’s happening*.

To keep it independent (so people can attend without session 1):

* Use a different story than payments/orders. For example:

  * “User profile service calls 3 downstreams → one is slow → cascades → outage.”
* Cover a practical “starter kit”:

  * timeouts, retries (with jitter/backoff), circuit breakers, bulkheads, rate limiting
  * then observability pillars (logs/metrics/traces) and what questions each answers

Hands-on exercise idea:

* Give them a “mystery latency spike” scenario and ask:

  1. what metric would reveal it fastest?
  2. what log fields are missing?
  3. what trace span would isolate the dependency?
  4. what resiliency control would prevent it next time?

## Make the independence obvious with a tiny framing tweak

Right now, both titles start with “mastering,” which is fine, but can sound “big.” For entry-level engineers, I’d consider lighter, outcome-based titles:

* **Idempotency & Eventual Consistency: Safe Retries and Async Systems (101)**
* **Resilience & Observability: Designing for Failure and Debugging Fast (101)**

## A simple “if you only take one” guide (helps sign-ups)

* Take **Session 1** if you work on APIs, queues, workflows, data pipelines, payments, provisioning.
* Take **Session 2** if you touch production services, on-call, incident response, performance, SLOs.

## One caution (worth handling explicitly)

Retries appear in *both* sessions:

* Session 1: retries as a *correctness* problem (idempotency)
* Session 2: retries as a *stability* problem (retry storms, timeouts, circuit breakers)

That’s totally fine—just call it out so it feels intentional, not repetitive.

If you want, I can sketch a tight 2-hour agenda for each (with timings, exercises, and a one-page takeaway checklist).

---

## Microservices 101 Series — Course Summary

Two entry-level, hands-on sessions that teach the “distributed-systems survival skills” behind reliable microservices. They share the same mental model—**assume timeouts, retries, and partial failure are normal**—but each session stands alone, so you can take either one (or both) without feeling lost.

**How to choose**

* Take **Session 1** if you work on **APIs, queues, workflows, data pipelines, payments, provisioning**.
* Take **Session 2** if you touch **production services, on-call, incident response, performance, SLOs**.

### One important note

Retries show up in both sessions, but with different intent:

* **Session 1:** retries as a **correctness** problem → *how to avoid duplicates* (idempotency)
* **Session 2:** retries as a **stability** problem → *how to avoid retry storms and cascades* (timeouts, controlled retry, circuit breakers)

---

## Microservices 101-01 (120 min)

**Title:** *Idempotency & Eventual Consistency — Safe Retries and Async Systems*

You’ll learn how real systems behave under uncertainty (timeouts, retries, duplicates) and how to design APIs and event consumers that stay correct—even when the same request/event happens more than once.

**Agenda**

* Introduction
* Idempotency 101
* **Exercise 1 (design) — 30 min**
* Eventual Consistency 101
* **Exercise 2 (scenario) — 30 min**
* Rules of thumb
* Wrap up

---

## Microservices 101-02 (120 min)

**Title:** *Resilience & Observability — Design for Failure and Debug Fast*

You’ll learn how slow dependencies turn into outages (cascades), how to contain blast radius with practical guardrails, and how to debug production issues quickly using metrics, traces, and logs—reducing MTTR.

**Agenda**

* Introduction
* Resilience starter kit (timeouts, controlled retry, breakers, bulkheads, degradation)
* **Exercise 1 (design) — 30 min**
* Observability 101 (practical)
* **Exercise 2 (debugging game) — 30 min**
* Wrap up
