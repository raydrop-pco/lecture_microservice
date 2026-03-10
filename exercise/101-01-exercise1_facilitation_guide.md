# Exercise 1 (30–35 min): Design a Retry-Safe “Create Order” API - Facilitation Guide

## Facilitation Guide (what you say / do)

### Timing

* 2 min: Explain scenario + constraints
* 10 min: Part A
* 10 min: Part B
* 10 min: Part C
* 5 min: Share-out + compare to model

### Prompts to circulate with

Ask teams:

* “What exactly is the *same logical action* here?”
* “What do you return on replay so it’s deterministic?”
* “Where is the ‘side-effect boundary’?”
* “What prevents two concurrent creates?”

### Common mistakes to watch for

* Using idempotency key but not scoping it (user collision)
* Only deduping in memory (breaks on restart)
* No concurrency control (race creates duplicates)
* Ignoring “same key, different body”
* TTL too short without reasoning

---

