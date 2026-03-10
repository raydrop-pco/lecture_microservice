# Session 2 (2h): Resilience & Observability — Design for Failure and Debug Fast (101)

## Outline

### Goals

* Prevent “one slow dependency kills everything.”
* Know the minimal resilience controls for production services.
* Use logs/metrics/traces to answer “what broke, where, and why?”

### Agenda (120 min)

**0–10: Hook + failure taxonomy**

* Latency > errors (slow is the new down)
* Cascading failure story

**10–35: Resilience starter kit**

* Timeouts: the #1 missing default
* Retries done right:

  * bounded retries
  * exponential backoff + jitter
  * retry budgets / limiting retry storms
* Circuit breakers: stop digging the hole
* Bulkheads: isolate blast radius
* Rate limiting / load shedding: survive overload
* Graceful degradation: partial results, cached responses

**35–55: Exercise 1 (design)**

* Service A calls B, C, D — C gets slow
* Ask them to choose:

  * timeout values
  * retry policy (what to retry, how many times)
  * breaker thresholds
  * fallback behavior

**55–80: Observability 101 (practical)**

* Metrics / logs / traces: what each is for
* The “golden signals” (latency, traffic, errors, saturation)
* Correlation IDs and structured logging
* Tracing basics: spans, service maps, critical path

**80–105: Exercise 2 (debugging game)**

* “p95 latency doubled, errors flat”
* Give them a small dataset/story and ask:

  1. first metric you check
  2. what tag/dimension you need (endpoint? region? dependency?)
  3. what log fields are missing
  4. what trace would confirm the culprit

**105–115: SLOs & alerting basics**

* Alert on symptoms (SLO burn) not causes (CPU 80%)
* What makes an alert actionable

**115–120: Wrap + 3 takeaways**

* Defaults matter: timeouts + bounded retries prevent most incidents
* Isolation beats heroics: bulkheads + breakers stop cascades
* Observability is a product feature for engineers

### Handout (1-pager you can share)

* “Production-ready defaults” table (timeouts, retries, breaker)
* Logging field checklist
* Metrics dashboard starter set
* Trace instrumentation checklist

---

## Session 2 — Slide Outline (Lecture Part)

### Slide - Cover

* **Speaker notes:** * “Session 1 was correctness under retries (idempotency, eventual consistency).”
* “Session 2 is stability and diagnosability: how to prevent slowdown from becoming outage, and how to shorten impact when it happens.”
* “This aligns with our internal resilience guidance: failure is normal; the goal is containment + fast recovery.” 

---

### Slide - Agenda

* **Speaker notes:** * “We’ll first build a mental model for cascades.”
* “Then the resilience starter kit (timeouts, controlled retry, breakers, bulkheads, load shedding).”
* “Then observability: metrics/logs/traces + a simple debug workflow.”
* “Exercises: one design, one debugging game.”

---

### Slide 1 — Title

* **Slide Title:** Resiliency & Observability 101
* **Subtitle / key message (optional):** *Design for failure; debug fast when it still happens.*
* **Slide contents:**

  * What you’ll learn:

    * Failure modes that cause outages (slowdown → cascade)
    * Resiliency “starter kit”: timeouts, controlled retry, circuit breakers, bulkheads
    * Observability basics: metrics/logs/traces + how to use them
  * What you’ll be able to do:

    * Choose sane defaults for **timeouts + controlled retry**
    * Identify cascades early and limit blast radius
    * Debug “latency spike” with a repeatable workflow
* **Diagram / illustration (optional):** Service A calling B/C/D with one dependency slow.
* **Speaker notes:** Position this session as **stability + diagnosability**; Session 1 was correctness under retry.

**Speaker notes**

* “Key idea: resilience is about preventing failure from spreading; observability is about reducing MTTR when failure happens.”
* “We’ll treat retry as a *controlled tool*, not a reliability hack.” 
* “And we’ll keep the Session 1 link explicit: retries are safe only if operations are idempotent.” 

---

### Slide 2 — Hook: “Slow is the new down”

* **Slide Title:** The Incident That Starts With Latency
* **Subtitle / key message:** *Most outages begin as a slowdown, not an error rate spike.*
* **Slide contents:**

  * Dependency gets slow → requests pile up
  * Queues grow / threads saturate
  * Timeouts trigger retries → more load → collapse
  * Slow dependency → **resource retention** → saturation
* **Diagram / illustration (optional):** Latency curve rising → saturation → failures.
* **Speaker notes:** Introduce the whitepaper’s “resource retention under stress” and cascade framing. 

* “Most outages begin as ‘it got slow’, not ‘it returned 500.’”
* “When a dependency slows down, requests stay in-flight longer.”
* “That’s resource retention: threads, connections, queue slots get held—then you hit saturation.” 
* “Once saturated, even healthy parts of the system can’t do work.”

---

### Slide 3 — Failure dynamics: retry amplification cascade

* **Slide Title:** Retry Amplification Cascade
* **Subtitle / key message:** *Retries can multiply traffic during a slowdown and accelerate collapse.*
* **Slide contents:**

  * When latency increases:

    * more requests overlap in-flight
    * more timeouts
    * more retries
  * Result: traffic multiplies, dependency worsens
* **Diagram / illustration (optional):** Feedback loop: Slow → Timeout → Retry → More load → Slower
* **Speaker notes:** Use the exact concept from the whitepaper (“retry amplification cascade”). 

* “This is the classic loop: latency rises → timeouts rise → retries rise → traffic multiplies → dependency worsens.”
* “Whitepaper names this explicitly: retry amplification cascade.” 
* “So retries must be controlled; otherwise they act like a load multiplier during stress.”

---

### Slide 4 — Timeout: the boundary (fail fast)

* **Slide Title:** Timeout Is a Boundary
* **Subtitle / key message:** *Without timeouts, failures spread; with timeouts, failures are contained.*
* **Slide contents:**

  * Timeouts prevent:

    * infinite waits
    * resource retention (threads, connections)
    * cascading stalls
  * Rule: set timeouts intentionally (never “infinite”)
* **Diagram / illustration (optional):** Call chain with explicit timeout budgets.
* **Speaker notes:** Reuse the whitepaper phrasing about timeouts as boundaries / fail-fast mechanism. 

* “Timeouts are the boundary that prevents waiting from consuming all your resources.”
* “Without timeouts, failures spread via resource retention; with timeouts, failures are contained.” 
* “A practical way to think: you have an end-to-end budget; every downstream call takes a slice.”
* “Never ‘infinite.’ Defaults should be explicit.”

---

### Slide 5 — Controlled retry: when, what, and how

* **Slide Title:** Controlled Retry (Not “Just Retry”)
* **Subtitle / key message:** *Retry is a tool; uncontrolled retry is a failure multiplier.*
* **Slide contents:**

  * Retry only on **transient** conditions (timeouts, 5xx, throttling)
  * Avoid retrying **persistent** failures (4xx, validation, known “down”)
  * Must include:

    * max attempts
    * exponential backoff
    * jitter
    * (Session 1) idempotency
* **Diagram / illustration (optional):** Backoff schedule with jitter.
* **Speaker notes:** Directly align with the whitepaper’s controlled retry section; explicitly reference Session 1 for idempotency safety.  

* “Controlled retry has four parts: **when** to retry, **how many**, **how spaced**, and **whether it’s safe**.”
* “Retry only transient conditions: timeouts, some 5xx, throttling.” 
* “Avoid persistent failures: validation/4xx, known-down dependencies.”
* “Bound attempts, use exponential backoff + jitter—this is exactly what our whitepaper recommends.” 
* “And link to Session 1: safe retry requires idempotency; otherwise you create duplicates.” 

---

### Slide 6 — Circuit breakers: stop digging

* **Slide Title:** Circuit Breaker
* **Subtitle / key message:** *When a dependency is failing/slow, stop calling it continuously.*
* **Slide contents:**

  * States: Closed → Open → Half-open

    * Closed: calls flow normally; failures are monitored
    * Open: calls are blocked (fail fast) for a cooldown period
    * Half-open: allow a small number of “probe” calls to test recovery

  * Benefits:

    * protects downstream
    * protects your own resources
    * speeds up recovery
* **Diagram / illustration (optional):** Circuit breaker state machine.
* **Speaker notes:** Explain with “fast failure” and recovery probing; keep configs conceptual (threshold, window, cooldown).

* “Circuit breakers prevent us from repeatedly calling a dependency that is clearly failing or timing out.”
* “Closed: normal flow; we monitor failures/latency.”
* “Open: fail fast for a cooldown—protects downstream and stops wasting our own threads/connections.”
* “Half-open: allow a few probe calls; if they succeed, close; if not, open again.”
* “This is a containment tool for cascades: it breaks the retry amplification loop.” 

  * “Circuit breakers turn slow/failing dependencies into fast failures, reducing resource retention and helping prevent cascading collapse.”
    * Closed: “Everything is normal. We allow requests through and keep rolling stats (error rate, timeouts, p95 latency). If failures cross a threshold, we trip.”
    * Open: “We stop calling the dependency for a short period. This protects the dependency and prevents our service from wasting threads/connections waiting on likely-failing calls.”
    * Half-open: “After cooldown, we cautiously try a few requests. If probes succeed, we close the breaker. If probes fail, we open again.”

---

### Slide 7 — Bulkheads and isolation

* **Slide Title:** Bulkheads: Limit Blast Radius
* **Subtitle / key message:** *Separate capacity pools so one failure doesn’t sink everything.*
* **Slide contents:**

  * Isolate by:

    * dependency (separate connection pools)
    * endpoint (separate thread pools / queues)
    * tenant / priority
  * Pair with graceful degradation
* **Diagram / illustration (optional):** Two compartments (“bulkheads”) with separate pools.
* **Speaker notes:** Tie back to “resource retention” idea: isolation prevents full saturation. 

* “Bulkheads are isolation: separate capacity pools so one dependency can’t consume everything.”
* “Examples: separate connection pools per dependency; separate thread pools per endpoint; separate queues for priority.”
* “This directly addresses the resource retention problem: even if one area stalls, others keep operating.” 

---

### Slide 8 — Load shedding & graceful degradation

* **Slide Title:** Shed Load, Degrade Gracefully
* **Subtitle / key message:** *In overload, doing less is how you survive.*
* **Slide contents:**

  * Rate limiting, queue caps, rejecting early
  * Degrade responses:

    * cached/stale data
    * partial results
    * “try again later”
* **Diagram / illustration (optional):** “critical path” kept, “nice-to-have” dropped.
* **Speaker notes:** Make this practical: “What’s your service’s ‘must keep’ behavior?”

* “In overload, doing less is how you survive.”
* “Load shedding: reject early before you saturate deeper layers—rate limits, queue caps, concurrency limits.”
* “Degradation: cached/stale, partial results, or a clear ‘try later’ response.”
* “Goal: keep core functionality alive and reduce time-to-recovery.”

---

### Slide 9 — Resilience summary: the minimal starter kit

* **Slide Title:** Production Defaults Starter Kit
* **Subtitle / key message:** *If you do only a few things, do these.*
* **Slide contents:**

  * Timeouts everywhere
  * Controlled retry (bounded + backoff + jitter)
  * Circuit breakers on remote calls
  * Bulkheads for critical dependencies
  * Clear fallback/degrade strategy
* **Diagram / illustration (optional):** Checklist icon set.
* **Speaker notes:** Reinforce: “You can’t prevent failure; you can prevent failure from spreading.”

* “If you do only a few things, do these.”
* “Timeouts to prevent stalls from spreading.” 
* “Controlled retry to avoid amplification.” 
* “Breakers + bulkheads to contain blast radius.”
* “Fallback strategy so users get something useful during partial failure.”

---

### Exercise 1

* **Slide Title:** Exercise 1: Design Resilience Controls for a Slow Dependency
* **Slide contents:**

  * Design resilience controls for a service experiencing a slow dependency.
  * **Focus**

    * Timeouts + **controlled retry** + circuit breaker + bulkheads
    * Load shedding + graceful degradation
    * *(Assume correctness/idempotency foundations from Session 1.)*
  * **Scenario**

    * Service **A** handles user requests and calls downstream **B**, **C**, and **D**.

      * Assume A calls B/C/D **in parallel** and waits for all three (common fan-out).
    * **C becomes slow** (latency increases; errors may stay flat).

      * Under load, A risks **resource saturation** and **cascading failure**.

* **Speaker notes:**

  * “This exercise bridges Session 1 and Session 2.”
  * “From Session 1: **idempotency makes retries safe**.”
  * “From today: **controlled retry** means bounded attempts + exponential backoff + jitter—always with explicit timeouts.”
  * “Your job: propose the guardrails that stop a slowdown from turning into an outage: timeouts, retries, breakers, bulkheads, shedding, and a clear fallback.”
  * “Even if A called B/C/D sequentially, the same controls apply—parallel fan-out just makes the saturation effect easier to visualize.”


---

### Slide 10 — Observability: why we need it

* **Slide Title:** Observability = Answering Questions Quickly
* **Subtitle / key message:** *When something breaks, you need to know where and why—fast.*
* **Slide contents:**

  * The three pillars:

    * Metrics (what/how much)
    * Logs (what happened)
    * Traces (where time went)
  * Observability reduces MTTR
    * MTTR (Mean Time To Restore): average time from incident start → service restored (or customer impact stopped)
* **Diagram / illustration (optional):** Metrics/Logs/Traces triangle.
* **Speaker notes:** Keep it operational: “We want a repeatable debugging path.”

  * “People often mix these up: **MTTF** is how long until something fails; **MTTR** is how fast we recover after it fails.”
  * “Observability mostly improves MTTR—it helps you find the cause and mitigation faster.”
  * “Reliability improves when failures are rarer (higher MTTF) and recovery is faster (lower MTTR).”

* “Observability is about shortening the time from ‘users complain’ to ‘we mitigated.’”
* “MTTF is how long until failure; MTTR is how fast we restore after failure.”
* “Observability mostly improves MTTR: faster detection, diagnosis, and safer mitigation.” (Your definition on-slide is good.) 

---

### Appendix — Reliability Metrics 101

* **Slide Title:** Appendix: MTTF vs MTTR (and why we care)
* **Subtitle / key message:** *Reliability = fewer failures + faster recovery.*
* **Slide contents:**

  * **MTTF (Mean Time To Failure):** average time between failures (higher is better)
  * **MTTR (Mean Time To Restore/Recover):** average time to restore service after failure (lower is better)
  * **Availability (intuition):** improves when **MTTF ↑** and **MTTR ↓**
  * Observability impact:

    * Mostly **reduces MTTR** (faster detection + diagnosis + mitigation)
* **Diagram / illustration (optional):** Simple timeline:

  * “up time” segment → “incident” segment (restore point) → “up time”
  * label long “incident” as high MTTR
* **Speaker notes:**

  * “If failures will happen anyway, your best lever is often MTTR.”
  * “This is why we invest in dashboards, traces, and structured logs: not to prevent all failures, but to shorten the impact window.”
  * Thia page must help you. https://newrelic.com/jp/blog/observability/what-is-mttr

---

### Slide 11 — Metrics that matter: golden signals

* **Slide Title:** Golden Signals
* **Subtitle / key message:** *Latency, Traffic, Errors, Saturation.*
* **Slide contents:**

  * Latency: p50/p95/p99
  * Traffic: RPS, concurrency
  * Errors: rate + type
  * Saturation: CPU, memory, queue depth, thread pool, connection pool
* **Diagram / illustration (optional):** Dashboard sketch with the four.
* **Speaker notes:** Emphasize: “Latency ↑ with errors flat is common—don’t wait for errors.”

* “These four signals catch most incidents early: latency, traffic, errors, saturation.”
* “Important: latency can spike while errors stay flat—don’t wait for 500s.”
* “Always slice by dimensions: endpoint, region, dependency, tenant—otherwise averages hide pain.”

---

### Slide 12 — Logs: structured and correlated

* **Slide Title:** Logs That Help (Not Logs That Hurt)
* **Subtitle / key message:** *Make logs queryable and linkable.*
* **Slide contents:**

  * Structured logging (JSON fields)
  * Must-have fields:

    * request_id / trace_id
    * user/tenant (safe)
    * endpoint, status code, latency
    * upstream/downstream dependency name + error code
* **Diagram / illustration (optional):** Example log entry with highlighted fields.
* **Speaker notes:** Explain “you can’t grep your way out of a distributed incident.”

* “Logs should be queryable, not just readable.”
* “Structured logs with correlation IDs let you connect a user symptom to a trace and a downstream error.”
* “Minimum useful set: request_id/trace_id, endpoint, status, latency, dependency name, error code, and safe tenant/user identifier.”

---

### Slide 13 — Traces: find the critical path

* **Slide Title:** Distributed Tracing
* **Subtitle / key message:** *Traces show where time went across services.*
* **Slide contents:**

  * Spans, parent/child, service map
  * Critical path vs parallel work
  * Common use: “Which dependency added p95?”
* **Diagram / illustration (optional):** Trace waterfall.
* **Speaker notes:** Tie to resilience controls: traces often reveal missing timeouts / retries / queueing delays.

* “Traces show where time went across services—especially for p95/p99.”
* “They make parallel vs sequential visible: critical path vs parallel spans.”
* “Great for questions like: ‘Which dependency added p95?’ and ‘Is the delay queueing or network or downstream compute?’”

---

### Slide 14 — Debug workflow: a simple playbook

* **Slide Title:** A Repeatable Debugging Workflow
* **Subtitle / key message:** *Start broad → narrow fast.*
* **Slide contents:**

  1. Confirm symptom (which SLO / which endpoint)
  2. Check golden signals + dimensions (region, instance, dependency)
  3. Jump to traces for slow requests
  4. Use logs to confirm error details and context
  5. Decide mitigation (shed load, disable feature, breaker open, rollback)
* **Diagram / illustration (optional):** Funnel from metrics → traces → logs.
* **Speaker notes:** This slide sets up Exercise 2 (debugging game).

* “This is the playbook: start broad, narrow quickly.”
* Step 1: “Confirm the symptom—what SLO, what endpoint, what time window.”
* Step 2: “Golden signals + dimensions to find where it’s concentrated.”
* Step 3: “Use traces to identify the slow span / dependency.”
* Step 4: “Use logs for exact error context.”
* Step 5: “Choose mitigation: shed load, degrade feature, open breaker, rollback.”
* “We’ll apply this in Exercise 2.”

---

### Exercise 2

* **Slide Title:** Exercise 2: Debugging Game — p95 Latency Spike

* **Slide contents:**

  * Debug a **p95 latency spike** using **metrics, traces, and logs** (to reduce **MTTR**).
  * Use the funnel: **metrics → slice → traces → logs → mitigation**
  * **Incident snapshot**

    * Symptom: A’s **p95 latency** doubled in the last **20 minutes**
    * Errors: mostly flat (no big 5xx spike)
    * Traffic: steady
    * Suspect: a downstream dependency is slower → **saturation / resource retention**

* **Speaker notes (ready-to-talk scenario):**

  * “We’re going to run this like a mini on-call incident. The goal isn’t to guess the exact root cause immediately—it’s to use a repeatable method.”
  * “Set the scene: it’s mid-day, users say the app ‘feels slow,’ dashboards show p95 jumped, but error rate looks normal.”
  * “This is the classic trap: if you wait for errors, you’ll be late. Latency + saturation are often the first signals.”
  * “Your task is to follow the funnel:

    1. **Metrics:** confirm the symptom and find where it concentrates (endpoint / region / dependency).
    2. **Slice:** break the averages—identify which slice explains the p95 jump.
    3. **Traces:** for slow requests, find the **critical path** span (where time went).
    4. **Logs:** confirm details (timeout vs slow response, retry count, breaker state, throttling codes).
    5. **Mitigation:** pick the fastest safe action to reduce customer impact (degrade, shed load, open breaker, rollback).”
  * “Assume the most likely story: one dependency got slower, requests stayed in-flight longer, resources piled up—**resource retention**—and the system is heading toward saturation.”
  * “We’ll do a quick share-out at the end: what you checked first, what evidence convinced you, and what mitigation you chose.”

---

### Slide 15 — Bridge back to design

* **Slide Title:** Design + Observe + Improve
* **Subtitle / key message:** *Resilience controls are only as good as your visibility.*
* **Slide contents:**

  * Controls (timeouts/retry/breakers/bulkheads) prevent cascades
  * Observability proves whether they work
  * Post-incident: add guardrails + add signals
* **Diagram / illustration (optional):** Loop: Build → Observe → Learn → Improve
* **Speaker notes:** Close the lecture: “Now we’ll apply this to a slow-dependency scenario and debug using metrics/logs/traces.”

* “Resilience controls prevent cascades; observability proves whether they’re working.”
* “Post-incident improvements usually fall into two buckets: add guardrails (timeouts/breakers/bulkheads) and add signals (dashboards/traces/log fields).”
* “Ownership means operability: if we ship it, we also make it diagnosable and safe under stress.”

---
