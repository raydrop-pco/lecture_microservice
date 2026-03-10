# Exercise 2 (30–35 min): Design an Eventually Consistent Order Workflow - Facilitation Guide

## Facilitation Guide (for you)

### Timing (recommended)

* 2 min intro: remind “design for stale / duplicate / out-of-order”
* 10 min Part A
* 10 min Part B
* 10 min Part C
* 5 min Part D + share-out

### Prompts to circulate with

* “What’s your single source of truth for order state?”
* “What does ‘in progress’ look like in your API?”
* “How do you prevent a duplicate event from advancing state twice?”
* “If events arrive out of order, what prevents impossible states?”

### Common mistakes to watch for

* No explicit “processing” states → client confusion
* Assuming event ordering
* Deduping in memory (fails on restart)
* Accepting any event at any time (state jumps)
* No story for partial success (payment succeeded, inventory failed)

---

