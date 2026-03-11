# Session 2 Handout — Resilience & Observability (1-pager)
Aligned guidance: prevent **resource retention** and **retry amplification**; contain blast radius; reduce **MTTR** via metrics/logs/traces.

---

## 1) Resilience starter kit (defaults)
- **Timeouts everywhere** (boundary; no infinite waits)
- **Controlled retry:** transient only; bounded attempts; exponential backoff + **jitter**
- **Safe retry:** retry only idempotent operations (idempotency keys/dedup)
- **Circuit breakers:** fail fast; half-open probes for recovery
- **Bulkheads:** isolate capacity pools to limit blast radius
- **Load shedding + degradation:** cap concurrency/queues; serve cached/stale/partial when possible

---

## 2) Quick “production defaults” template
| Control | Default you pick |
|---|---|
| End-to-end timeout budget | _____ ms |
| Downstream call timeout | _____ ms |
| Retries (transient only) | Max _____ attempts; backoff _____; jitter yes/no |
| Circuit breaker | Trip: _____ ; Cooldown: _____ ; Probes: _____ |
| Bulkhead | Isolate by _____ ; Pool size _____ |
| Load shedding | Trigger: _____ ; Action: reject/queue cap/degrade |

---

## 3) Observability reduces MTTR
- **MTTF** = time until failure  
- **MTTR** = time to restore after failure  
Observability mainly improves **MTTR**.

**Debug funnel**
1) confirm symptom  
2) golden signals + slice  
3) traces (critical path)  
4) logs (confirm)  
5) mitigate + verify
