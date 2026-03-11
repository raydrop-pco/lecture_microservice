# Session 2 — Exercise 1 Model Answer (Share After)
Reference solution (illustrative). What matters: prevent **resource retention** and **retry amplification**; contain blast radius; degrade intentionally.

---

## A) Timeouts + Controlled Retry (example)
**Timeouts:** explicit budgets. Example end-to-end 800ms; A->B 200ms, A->C 250ms, A->D 200ms (leave margin).  
**Controlled retry (transient only):**
- max 2 attempts total (1 retry)
- exponential backoff (e.g., 50ms then 150ms)
- **jitter** on
- never retry validation/4xx

**Safe retry:** retry only idempotent operations; otherwise add idempotency keys/dedup before enabling retry.

---

## B) Circuit Breaker + Bulkheads (example)
**Breaker for C:**
- trip on timeout rate or high latency (e.g., >30% timeouts over 10s)
- open 30-60s cooldown
- half-open: small probe rate; close on success, reopen on failure

**Bulkheads:**
- isolate pools per dependency so slow C cannot starve calls to B/D or request handling
- optionally isolate by endpoint/priority

---

## C) Degrade + Shed Load (example)
**Degrade:** return cached/stale or partial response with a clear flag; if none, fail fast with 503 + Retry-After.  
**Load shedding:** cap concurrency/queue depth; reject early when saturation triggers.

---

## D) Signals (example set)
- A latency p95/p99 by endpoint
- dependency C latency + timeout rate
- retry rate (watch amplification)
- saturation (thread/conn pools, queue depth)
- breaker state

Read these both ways: check each signal individually, then interpret them together as one system story.
