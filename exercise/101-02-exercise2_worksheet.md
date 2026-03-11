# Session 2 — Exercise 2 Worksheet (1-pager)
Debugging game: **p95 latency doubled while errors stayed flat**.  
Use the funnel: **metrics -> slice -> traces -> logs -> mitigation**.

---

## Incident snapshot
- Symptom: A's **p95 latency** doubled in the last 20 minutes
- Errors: mostly flat (no big 5xx spike)
- Traffic: steady
- Suspect: a downstream dependency is slower -> saturation/resource retention

---

## Part A — Start with metrics (10 min)
1) Which **golden signals** do you check first? List top 3 graphs.  
2) What **dimensions** do you slice by? (endpoint, region, instance, dependency, tenant...)
3) What does p95 mean in this incident context, and why is p50 alone not enough?

Important: check each signal individually, then combine them to determine the most likely failure mode.

---

## Part B — Use traces (10 min)
4) What trace question are you trying to answer?  
   Example: "Which span dominates the critical path at p95?"  
5) If dependency C is slow, what do you inspect next?
- retries?
- queueing/wait time?
- near-timeout behavior?

---

## Part C — Confirm with logs (8 min)
6) What log fields do you need to confirm quickly?
- trace_id/request_id
- dependency
- timeout vs error
- retry_count
- breaker_state
- throttling code
- endpoint, latency

---

## Part D — Mitigate (5 min)
7) Choose one mitigation and justify:
- shed load
- open breaker
- disable feature / degrade
- rollback
- scale (careful)
- increase timeouts (rare)
