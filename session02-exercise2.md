# Session 2 — Exercise 2 Worksheet (1-pager)
Debugging game: **p95 latency doubled while errors stayed flat**.  
Use the funnel: **metrics → slice → traces → logs → mitigation**.

---

## Incident snapshot
- Symptom: A’s **p95 latency** doubled in the last 20 minutes
- Errors: mostly flat (no big 5xx spike)
- Traffic: steady
- Suspect: a downstream dependency is slower → saturation/resource retention

---

## Part A — Start with metrics (10 min)
1) Which **golden signals** do you check first? List top 3 graphs.  
2) What **dimensions** do you slice by? (endpoint, region, instance, dependency, tenant…)
3) What does p95 mean in this incident context, and why is p50 alone not enough?

Important: check each signal individually, then combine them to determine the most likely failure mode.

---

## Part B — Use traces (10 min)
4) What trace question are you trying to answer?  
   Example: “Which span dominates the critical path at p95?”  
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

---

---

# Session 2 — Exercise 2 Facilitation Guide
Objective: practice a calm, repeatable debugging funnel and choose mitigations that reduce impact quickly (without worsening resource retention).

---

## Timing (30–35 min)
- 2 min intro + roles (IC, metrics, traces/logs)
- 10 min metrics + slicing
- 10 min traces reasoning
- 8 min logs confirmation
- 5 min mitigation + share-out

---

## Prompts
- “Latency up, errors flat—what does saturation look like?”
- “Which dependency’s p95 changed?”
- “Do traces show queueing vs downstream compute vs retries?”
- “What’s the fastest safe mitigation?”
- “Which dimension slice reveals where customer pain is concentrated?”
- “What pattern do the four signals tell together, not individually?”

---

## Pitfalls
- Looking only at averages (p50) vs tail (p95/p99)
- Not slicing by endpoint/dependency/region
- Jumping to logs first (noise) vs metrics → traces
- Blindly increasing timeouts (can worsen retention)

---

---

# Session 2 — Exercise 2 Answer Sheet (Share After)
Reference: method > exact root cause. Show the funnel.

---

## A) Metrics
- Check A latency p95/p99 and saturation (thread/conn pools, queue depth)
- Slice by endpoint/region/dependency
- Check dependency C latency + timeout rate
- Check retry rate (amplification indicator)
- Interpret signals together (latency, traffic, errors, saturation) before jumping to a cause.

---

## B) Traces
- Open traces for slow requests (p95 cohort)
- Identify the span dominating the critical path
- If C dominates, inspect retries, queueing/wait time, proximity to timeout

---

## C) Logs
- Filter logs by trace_id/request_id
- Confirm dependency=C, outcome=timeout/slow, retry_count, breaker_state, throttling codes
- Use structured fields to aggregate by dependency/endpoints

---

## D) Mitigations (examples)
Prefer mitigations that reduce load/impact:
- open/strengthen breaker for C
- degrade feature using C (cached/stale/partial)
- shed load (cap concurrency)
- rollback a change affecting C
Avoid raising timeouts unless capacity margin is proven.

---
