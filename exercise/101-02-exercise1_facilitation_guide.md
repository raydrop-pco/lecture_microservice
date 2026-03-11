# Session 2 — Exercise 1 Facilitation Guide
Objective: design controls that stop a slowdown from becoming an outage:
**timeouts (boundary) + controlled retry (bounded/backoff/jitter) + breakers/bulkheads + load shedding + clear fallback**.

---

## Timing (30–35 min)
- 2 min intro (slowdown -> resource retention -> saturation -> retry amplification)
- 10 min Part A (timeouts + controlled retry + idempotency assumptions)
- 10 min Part B (breaker + bulkheads)
- 8 min Part C (degrade + shed load)
- 5 min Part D (signals) + share-out

---

## Prompts to ask
- "Where does the time go when C is slow-threads, connections, queues?"
- "How do you prevent retry storms?"
- "How do you fail fast when C is unhealthy?"
- "What must remain available vs what can degrade?"
- "Which metrics are you reading individually, and what story do they tell when combined?"
- "Which dimensions will you slice first (endpoint, region, dependency, tenant)?"

---

## Common pitfalls
- Missing timeouts or timeouts > end-to-end SLA
- Unbounded retries or no jitter
- Breaker without half-open probes
- No isolation (one dependency saturates everything)
- No clear degraded behavior
- Looking at averages only and missing localized tail latency
