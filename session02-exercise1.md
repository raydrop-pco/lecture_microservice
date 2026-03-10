# Session 2 — Exercise 1 Worksheet (1-pager)
Design resilience controls for a service experiencing a slow dependency.  
Focus: **timeouts + controlled retry + circuit breaker/bulkhead + load shedding + graceful degradation**.  
(Assume correctness/idempotency foundations from Session 1.)

---

## Scenario
Service **A** handles user requests and calls downstream **B**, **C**, and **D**.  
**C becomes slow** (latency increases; errors may stay flat). Under load, A risks **resource saturation** and **cascading failure**.

Assume A calls B/C/D in parallel and waits for all three (common fan-out).
This makes the “slow C causes resource retention + tail latency + saturation” story very clear.

---

## Part A — Choose safe defaults (10 min)
1) **Timeouts:** propose A→B, A→C, A→D timeout values and an end-to-end budget.  
2) **Controlled retry** (transient failures only: timeouts/5xx/throttling):  
   - max attempts = ___  
   - backoff = ___  
   - jitter = yes/no  
3) **Idempotency assumption:** which operations are safe to retry and why?  
   - If not safe, what must change?

---

## Part B — Contain blast radius (10 min)
4) **Circuit breaker** for C:
   - what trips it (error/timeout/latency)?
   - cooldown duration?
   - half-open probes (how many / how often)?
5) **Bulkheads:** what capacity pools are isolated (threads/connections/queues) and along which dimension?
   - dependency / endpoint / priority / tenant

---

## Part C — Degrade gracefully (8 min)
6) When C is slow/unavailable, what does A return?
- cached/stale
- partial result
- fallback default
- explicit “try later”
7) What is your **load shedding** rule (rate limit / queue cap / concurrency cap), and when does it trigger?

---

## Part D — What would you measure? (5 min)
List 5 signals to detect the cascade early:
- latency p95/p99
- saturation (thread/conn pools)
- queue depth
- downstream error/timeout rate
- retry rate
- breaker state
- etc.

---

---

# Session 2 — Exercise 1 Facilitator Cheat Sheet
Objective: design controls that stop a slowdown from becoming an outage:
**timeouts (boundary) + controlled retry (bounded/backoff/jitter) + breakers/bulkheads + load shedding + clear fallback**.

---

## Timing (30–35 min)
- 2 min intro (slowdown → resource retention → saturation → retry amplification)
- 10 min Part A (timeouts + controlled retry + idempotency assumptions)
- 10 min Part B (breaker + bulkheads)
- 8 min Part C (degrade + shed load)
- 5 min Part D (signals) + share-out

---

## Prompts to ask
- “Where does the time go when C is slow—threads, connections, queues?”
- “How do you prevent retry storms?”
- “How do you fail fast when C is unhealthy?”
- “What must remain available vs what can degrade?”

---

## Common pitfalls
- Missing timeouts or timeouts > end-to-end SLA
- Unbounded retries or no jitter
- Breaker without half-open probes
- No isolation (one dependency saturates everything)
- No clear degraded behavior

---

---

# Session 2 — Exercise 1 Answer Sheet (Share After)
Reference solution (illustrative). What matters: prevent **resource retention** and **retry amplification**; contain blast radius; degrade intentionally.

---

## A) Timeouts + controlled retry (example)
**Timeouts:** explicit budgets. Example end-to-end 800ms; A→B 200ms, A→C 250ms, A→D 200ms (leave margin).  
**Controlled retry (transient only):**
- max 2 attempts total (1 retry)
- exponential backoff (e.g., 50ms then 150ms)
- **jitter** on
- never retry validation/4xx

**Safe retry:** retry only idempotent operations; otherwise add idempotency keys/dedup before enabling retry.

---

## B) Circuit breaker + bulkheads (example)
**Breaker for C:**
- trip on timeout rate or high latency (e.g., >30% timeouts over 10s)
- open 30–60s cooldown
- half-open: small probe rate; close on success, reopen on failure

**Bulkheads:**
- isolate pools per dependency so slow C cannot starve calls to B/D or request handling
- optionally isolate by endpoint/priority

---

## C) Degrade + shed load (example)
**Degrade:** return cached/stale or partial response with a clear flag; if none, fail fast with 503 + Retry-After.  
**Load shedding:** cap concurrency/queue depth; reject early when saturation triggers.

---

## D) Signals (example set)
- A latency p95/p99 by endpoint
- dependency C latency + timeout rate
- retry rate (watch amplification)
- saturation (thread/conn pools, queue depth)
- breaker state

---
