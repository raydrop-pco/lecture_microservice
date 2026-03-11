# Session 2 — Exercise 1 Worksheet (1-pager)
Design resilience controls for a service experiencing a slow dependency.  
Focus: **timeouts + controlled retry + circuit breaker + bulkheads + load shedding + graceful degradation**.  
(Assume correctness/idempotency foundations from Session 1.)

---

## Scenario
Service **A** handles user requests and calls downstream **B**, **C**, and **D**.  
**C becomes slow** (latency increases; errors may stay flat). Under load, A risks **resource saturation** and **cascading failure**.

Assume A calls B/C/D in parallel and waits for all three (common fan-out).
This makes the "slow C causes resource retention + tail latency + saturation" story very clear.

---

## Part A — Choose safe defaults (10 min)
1) **Timeouts:** propose A->B, A->C, A->D timeout values and an end-to-end budget.  
2) **Controlled retry** (transient failures only: timeout/5xx/throttling):  
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
5) **Bulkheads:** what capacity pools are isolated (threads/connections/queues), and along which dimension?
   - dependency / endpoint / priority / tenant

---

## Part C — Degrade gracefully (8 min)
6) When C is slow or unavailable, what does A return?
- cached/stale
- partial result
- fallback default
- explicit "try later"
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

Important: do not rely on global averages only. Slice by endpoint, region, dependency, and tenant to expose hidden pain.
