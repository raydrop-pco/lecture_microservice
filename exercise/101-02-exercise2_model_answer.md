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
