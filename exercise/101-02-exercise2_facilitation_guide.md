# Session 2 — Exercise 2 Facilitation Guide
Objective: practice a calm, repeatable debugging funnel and choose mitigations that reduce impact quickly (without worsening resource retention).

---

## Timing (30-35 min)
- 2 min intro + roles (IC, metrics, traces/logs)
- 10 min metrics + slicing
- 10 min traces reasoning
- 8 min logs confirmation
- 5 min mitigation + share-out

---

## Prompts
- "Latency up, errors flat-what does saturation look like?"
- "Which dependency's p95 changed?"
- "Do traces show queueing vs downstream compute vs retries?"
- "What's the fastest safe mitigation?"
- "Which dimension slice reveals where customer pain is concentrated?"
- "What pattern do the four signals tell together, not individually?"

---

## Pitfalls
- Looking only at averages (p50) vs tail (p95/p99)
- Not slicing by endpoint/dependency/region
- Jumping to logs first (noise) vs metrics -> traces
- Blindly increasing timeouts (can worsen retention)
