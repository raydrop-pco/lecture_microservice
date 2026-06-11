# Session 5 (2h): API Design for Evolution — Evolve Contracts Without Breaking Consumers

## Outline

### What a reader should be able to explain (clearly, to others)
* Why API evolution is dangerous in distributed systems: clients upgrade at different times, and some may not upgrade at all.
* The difference between a compatible change, an additive change, and a breaking change.
* Why API evolution is also a behavioral-coupling problem: clients depend on semantics, defaults, and error conventions, not only field names.
* Why the Expand & Contract pattern is safer than "just rename the field and deploy it."
* The practical contract differences between REST, gRPC, and GraphQL when evolving an API.
* The tradeoffs between API versioning strategies: URL path, header/media type, and schema-based evolution.

### Goals

* Define API evolution as a compatibility problem, not merely a code-change problem: the real question is whether old and new clients can keep working during and after the rollout.
* Carry forward Module 04's behavioral-coupling lens: changing response semantics, error handling, or default behavior can break consumers even when the schema barely changes.
* Teach a concrete migration playbook for breaking-looking changes using the Expand & Contract pattern, so engineers can rename fields, split payloads, or replace endpoints without forcing zero-downtime clients to break.
* Compare REST, gRPC, and GraphQL from an evolution perspective, focusing on how each style expresses contracts, what kinds of changes are usually safe, and where teams get surprised.
* Distinguish versioning strategies from compatibility strategies: version numbers can help routing and governance, but additive change discipline and deprecation policy are what actually prevent client outages.
* Connect API evolution to rollout safety: deprecation windows, observability, and feature flags are operational tools for proving a change is safe before old behavior is removed.

### Agenda (120 min)

**0–12: Hook + the blast-radius story**

* The incident: `user_id` becomes `account_id`, and an old mobile app crashes in production
* Why API changes are uniquely dangerous:
  * clients deploy on different schedules
  * third-party or mobile consumers may be outside your release control
  * old and new versions often coexist longer than expected
* **Compatibility is the default — but not a religion** *(we will come back to this)*
* One framing rule for the whole module:
  * your server can deploy today; your clients might not

**12–35: Compatibility 101 — what actually breaks clients**

* Contract thinking:
  * the API contract is what clients rely on, not what the backend team intended
  * compatibility means existing clients still work without modification
  * this is behavioral coupling at the API boundary: consumers start depending on semantics, defaults, and error conventions
* Change taxonomy:
  * additive changes: usually safe when clients ignore unknown fields
  * semantic changes: dangerous even if the schema looks the same
  * breaking changes: removed fields, renamed fields, required-field additions, changed response shape
* Common traps:
  * changing nullability or default behavior quietly
  * changing `200`/empty to `404` or altering error semantics without treating it as a contract change
  * reusing an old field name with new meaning
  * assuming all consumers read documentation before upgrading

**35–60: Expand & Contract — the zero-downtime migration pattern**

* Expand phase:
  * add the new field or behavior first
  * keep producing the old field in parallel
  * teach consumers to read both old and new shapes
* Transition phase:
  * measure adoption
  * communicate deprecation clearly
  * support mixed-version clients for a defined window
* Contract phase:
  * remove the old field only after evidence that consumers have migrated
* Walkthrough example:
  * rename `user_id` to `account_id` without breaking existing clients
  * server write/read behavior during each phase
  * how logs and metrics prove readiness for removal

**60–65: When to break the contract intentionally**

* Revisit the principle: compatibility is the default — but not a religion
  * Most of this module covers how to avoid breaking clients; that discipline is the foundation
  * Some changes cannot be made backward-compatible without compromising security, compliance, or long-term platform integrity
* Real-world precedents:
  * Google OAuth security tightening (2017–2022): short-term migration cost for all OAuth consumers, long-term security and ecosystem gain
  * LINE API platform overhaul (September 2016): replaced BOT API with Messaging API v2, forcing a full migration for all developers
* Criteria for a justified breaking change:
  * security or compliance requirement that cannot be met additively
  * elimination of critical technical debt that blocks platform evolution
  * a concrete migration path and deprecation window are provided
* The B2B SaaS dynamic:
  * influential customers may resist migration; that pressure must be weighed against long-term platform health
  * delaying a necessary break typically makes it more expensive and risky later
* Decision framing: breaking change is a last resort, not a design style

**65–80: API styles & evolution rules — REST vs gRPC vs GraphQL**

* REST:
  * where the contract lives: mostly implicit in behavior, supported by docs (OpenAPI/Swagger)
  * compatibility lens: easiest style to break accidentally because clients may rely on undocumented behavior
  * evolution rule of thumb: additive fields are often safe; removal/rename usually breaks someone
* gRPC:
  * where the contract lives: `.proto` files are the source of truth
  * compatibility lens: evolution rules are explicit guardrails (field numbers, optionality, reserved fields)
  * evolution rule of thumb: never reuse retired field numbers; additive fields are usually safe
* GraphQL:
  * where the contract lives: the GraphQL schema itself
  * compatibility lens: schema makes change impact visible, especially in tooling and IDEs
  * evolution rule of thumb: prefer deprecate-first workflow; mark fields deprecated before removal so clients get warnings
* Comparison question:
  * for your current stack, where is your real contract written today, and what makes compatibility failures most likely?

**80–95: Exercise 1 (design) — rename a critical field safely**

* Scenario: a widely used API response contains `user_id`, but the domain now requires `account_id`
* Ask participants to design:
  * the expand phase response shape
  * how long both fields stay available
  * what telemetry proves client migration
  * the exact rule for when the old field can be removed
* Debrief:
  * which client types are hardest to migrate?
  * what would make this plan unsafe?

**95–110: Versioning strategies + rollout safety**

* Versioning strategies:
  * URL path versioning (`/v1`, `/v2`)
  * header or media-type versioning
  * schema-first evolution without frequent explicit version bumps
* Tradeoffs:
  * explicit versions improve clarity, but create parallel maintenance cost
  * too many versions can hide poor compatibility discipline
  * not every change deserves a new version
* Rollout safety tools:
  * deprecation policy and sunset communication
  * consumer usage observability by version/field
  * feature flags or selective rollout to limit blast radius for risky behavior changes

**110–120: Exercise 2 + wrap**

* Exercise 2:
  * use a feature flag to expose a new response behavior to 10% of traffic first
  * decide what success and rollback signals look like
* Wrap + 3 takeaways:
  * compatibility is a runtime concern, not just a schema concern
  * always expand before you contract
  * versioning helps, but disciplined compatibility rules prevent outages

### Handout (1-pager you can share)

* Compatibility checklist: safe vs risky vs breaking changes
* Expand & Contract migration playbook
* REST / gRPC / GraphQL evolution rules-of-thumb table
* Versioning strategy comparison and deprecation policy template

---

## Session 5 — Slide Outline (Lecture Part)

### Slide 1 — Cover

**Slide Title:** Microservices 101-05: API Design for Evolution
**Subtitle / key message:** *Evolve Contracts Without Breaking Consumers*
**Slide contents:**
* (Speaker introduces the session, title slide only)

**Speaker notes:**
* "Welcome to Session 5 of Microservices 101."
* "Today is about evolution: how APIs change over time, and what it takes to change them without breaking the clients that depend on them."
* "We'll build both a vocabulary and a concrete migration playbook."

---

### Slide 2 — Introduction of This Series

**Slide Title:** Microservices 101 Series
**Slide contents:**
* 101-01 Idempotency & Eventual Consistency — Safe Retries and Async Systems
* 101-02 Resilience & Observability — Design for Failure and Debug Fast
* 101-03 Data Boundaries & Ownership — Own the domain and move data safely
* 101-04 Workflows & Messaging — Coordination for Loose Coupling
* **101-05 API Design for Evolution — Evolve contracts without breaking consumers**
* 101-06 Deployment Safety — Ship changes safely with canary/blue-green and feature flags

**Speaker notes:**
* "Session 1 was correctness under retries: idempotency and eventual consistency."
* "Session 2 was stability and diagnosability: resilience controls and observability."
* "Session 3 was data: ownership boundaries, outbox pattern, and query patterns for split data."
* "Session 4 was coordination: communication levels, orchestration, and choreography."
* "Today we zoom in on the contract between services: what it means to change an API in a distributed system where clients do not upgrade on your schedule."

---

### Slide 3 — Agenda

**Slide Title:** Agenda
**Slide contents:**
* Hook — the blast-radius story
* Compatibility 101 — what actually breaks clients
* Expand & Contract — the zero-downtime migration pattern
* When to break the contract intentionally
* API Styles & Evolution Rules — REST vs gRPC vs GraphQL
* Exercise 1 — rename a critical field safely [15 min]
* Versioning Strategies + Rollout Safety
* Exercise 2 + Wrap [10 min]

**Speaker notes:**
* "We open with an incident story: a field rename that crashed an old mobile app in production."
* "Then we build vocabulary: what a contract is, what kinds of changes break clients, and the common traps."
* "The core technique is Expand & Contract — a three-phase migration playbook for making breaking-looking changes safely."
* "We'll also address a question beginners often get wrong: is a breaking change always a failure? The short answer is no — but it requires specific criteria and process."
* "The API styles section shows how compatibility looks different in REST, gRPC, and GraphQL because the contract lives in a different place in each."

---

### Slide 4 — Key Takeaways

**Slide Title:** API Design for Evolution
**Subtitle / key message:** *Evolve contracts without breaking consumers.*

**Slide contents:**
* What you'll learn:
  * Why API evolution is uniquely dangerous in distributed systems
  * The difference between compatible, additive, and breaking changes
  * The Expand & Contract migration pattern
  * How REST, gRPC, and GraphQL each express and evolve their contracts
* What you'll be able to do:
  * Classify a proposed change as safe, risky, or breaking
  * Design a safe field-rename migration using Expand & Contract
  * Choose an appropriate versioning and deprecation strategy

**Speaker notes:**
* "After this session you'll be able to explain API evolution as a compatibility problem — not just a code-change problem."
* "The real question is never 'did we change the code?' — it's 'can old and new clients still work during and after the rollout?'"
* "Everything today flows from that one question."

---

### Slide 5 — Divider: Hook

---

### Slide 6 — The Incident

**Slide Title:** The Day `user_id` Became `account_id`
**Subtitle / key message:** *A one-line rename. A production crash. An old mobile app that didn't get the memo.*

**Slide contents:**
* Backend team renames the field in the API response
* New mobile app version reads `account_id` — works fine
* Old mobile app (still in the wild) reads `user_id` — gets `null`
* User profile fails to load; crash reported
* Root cause: the server deployed today; the old client did not

```
GET /api/profile

── Before deploy ────────────────────  ── After deploy ─────────────────────
{                                      {
  "user_id":  "u-123",       ◀ gone      "account_id": "u-123",   ◀ new
  "name":     "John Doe"               "name":       "John Doe"
}                                      }

Old client code:                       Old client code (unchanged):
  val id = response["user_id"]           val id = response["user_id"]
  //→ "u-123" ✓                          //→ null  ✗  crash
```

**Speaker notes:**
* "This is the incident we'll return to throughout the session."
* "The change was small. The intent was reasonable. The blast radius was larger than expected."
* "This isn't an exotic edge case — it's the default situation in any system with mobile clients, third-party integrations, or multi-team consumers."
* "The lesson isn't 'never rename fields.' It's: a rename is a two-step operation in distributed systems, not one."

---

### Slide 7 — Why API Changes Are Uniquely Dangerous

**Slide Title:** Why API Changes Are Uniquely Dangerous
**Subtitle / key message:** *Your server deploys today. Your clients might not.*

**Slide contents:**
* Clients deploy on different schedules
  * Mobile apps require user installs; old versions linger in the wild
  * Third-party consumers may not even know a change is coming
* Consumers outside your release control
  * Partners, external developers, embedded devices
  * You cannot force them to upgrade
* Old and new versions coexist longer than expected
  * Rollouts are gradual; rollbacks are possible
  * "Deploy it and see" is the wrong mental model

**Speaker notes:**
* "In a monolith, you change the code and ship it — everything stays in sync."
* "In distributed systems, the consumer and producer are on separate release cycles."
* "The dangerous assumption is: 'our clients will upgrade quickly.' They won't, and some can't."
* "One framing rule to carry through the whole session: your server can deploy today; your clients might not."

---

### Slide 8 — Framing Rule

**Slide Title:** Compatibility Is the Default — But Not a Religion
**Subtitle / key message:** *Most of this session is about maintaining compatibility. But we'll also cover when breaking it is the right call.*

**Slide contents:**
* Default rule: design every change to keep existing clients working
* The key question: can old and new clients still work during and after the rollout?
* We will also ask: when is a breaking change strategically necessary?
  * *(we will return to this in the "When to break the contract intentionally" section)*

**Speaker notes:**
* "Beginners sometimes take the wrong lesson: 'breaking change = failure, always.'"
* "That's too simple. The real rule is: compatibility is the default discipline; breaking is a last resort with specific criteria and process."
* "We'll build the vocabulary first, then the technique, and then revisit the exception."

---

### Slide 9 — Divider: Compatibility 101

---

### Slide 10 — What Is an API Contract?

**Slide Title:** The API Contract: What Clients Actually Depend On
**Subtitle / key message:** *The contract is what clients rely on — not what the backend team intended.*

**Slide contents:**
* Contracts include (explicitly or implicitly):
  * Field names and types in request and response
  * Response shape and nesting
  * Nullability and default values
  * Error codes and error response structure
  * Response semantics such as "empty result" vs "not found"
  * Endpoint URL and HTTP method (REST)
  * Field numbers (gRPC / Protobuf)
  * Type definitions (GraphQL schema)
* Compatibility means: existing clients still work without modification

**Speaker notes:**
* "A contract is not just the OpenAPI spec or the Protobuf file. It includes every behavior the client has observed and started relying on."
* "That includes undocumented behavior: the fact that a field was never null, even if the spec said it was nullable."
* "This is the same idea we called behavioral coupling in Module 04: the caller has coupled itself to the callee's runtime behavior, not just its documented schema."
* "This is why compatibility is harder than it looks — you're not just responsible for what you declared, but for what clients built on top of."

---

### Slide 11 — Change Taxonomy: Safe, Risky, Breaking

**Slide Title:** Three Kinds of Changes
**Subtitle / key message:** *Not all changes are equal — and the dangerous ones are not always obvious.*

**Slide contents:**
* **Additive changes** — usually safe
  * Add a new optional field to the response
  * Add a new endpoint
  * Add a new enum value *(be careful: some clients switch on enums)*
* **Semantic changes** — dangerous even if the schema looks the same
  * Same field name, different meaning
  * Changed nullability behavior
  * Changed default value
  * Changed error semantics or status-code meaning
* **Breaking changes** — existing clients stop working
  * Remove a field
  * Rename a field
  * Add a required field to a request
  * Change a response shape

**Speaker notes:**
* "The taxonomy is important because teams often focus only on breaking changes and miss semantic drift."
* "A field that was always non-null becoming nullable is a semantic change: the schema may not change, but the client assumption breaks."
* "Likewise, returning `404` where clients learned to interpret `200` with an empty payload as 'not found' is a behavioral contract change, even if the endpoint and schema look familiar."
* "A renamed field looks like a clean two-line diff; in a distributed system it's a multi-phase operation."
* "Quick check: which category does our `user_id` → `account_id` rename fall into? Breaking — the old field disappears."

---

### Slide 12 — Common Traps

**Slide Title:** Common Compatibility Traps
**Subtitle / key message:** *The most dangerous changes are the ones that feel safe.*

**Slide contents:**
* **Changing nullability quietly**
  * A field that was always non-null becomes sometimes-null — client crashes on null dereference
* **Changing response semantics behind the same endpoint**
  * Yesterday `200 {}` meant "not found"; today `404` means "not found" — clients now misclassify a valid business case as a system error
* **Reusing an old field name with new meaning**
  * Client reads stale semantics; silent data corruption
* **Assuming all consumers read documentation before upgrading**
  * Many consumers sniff behavior, not spec; real-world behavior is the contract
* **Semantic drift behind a stable endpoint**
  * Same URL, same field names — but the meaning of the values has shifted

**Speaker notes:**
* "These are the traps that are hardest to catch in code review because the schema diff looks clean."
* "Nullability change: the CI build passes; the mobile client crashes in production three days later."
* "The status-code example should feel familiar from Module 04: this is behavioral coupling showing up as compatibility failure."
* "Field-name reuse: if you remove `status` and add `status` back with a different meaning, clients reading the old meaning get corrupted data silently."
* "The takeaway: the contract is what clients observe, not what you documented."

---

### Slide 13 — The Expand & Contract Pattern

**Slide Title:** Expand & Contract: The Three-Phase Migration
**Subtitle / key message:** *Never remove before you add. Never remove before you've confirmed clients have moved.*

**Slide contents:**

```
Phase 1 — EXPAND         Phase 2 — TRANSITION      Phase 3 — CONTRACT

Server produces:         Server produces:           Server produces:
  user_id  ← old          user_id  ← old              account_id ← new
  account_id ← new        account_id ← new
                                                     Old clients migrated ✓
Old client:   reads       Old client:  migrates      user_id removed safely
  user_id ✓               → account_id ✓
New client:   reads
  account_id ✓
```

* Expand: add the new field; keep the old field producing in parallel
* Transition: update consumers to read the new field; monitor migration
* Contract: remove the old field only after evidence that all consumers have moved

**Speaker notes:**
* "The key insight: a field rename in a distributed system is not one operation — it's three phases over time."
* "Each phase is independently deployable. The server can move independently of the clients."
* "This is where the session title becomes concrete: you can evolve the contract without breaking consumers."

---

### Slide 14 — Expand Phase

**Slide Title:** Phase 1: Expand
**Subtitle / key message:** *Add the new. Keep the old. Both work simultaneously.*

**Slide contents:**
* Server change:
  * Start producing the new field (`account_id`) alongside the old (`user_id`)
  * Both fields in every response — no client needs to change yet
* Client change (optional, parallel):
  * Teams can begin migrating to read the new field at their own pace
* Why this is safe:
  * Old clients still get the field they expect
  * New behavior is additive — no existing client breaks

**Speaker notes:**
* "The expand step is the one most teams skip — they rename the field and deploy. That's the mistake."
* "Adding both fields costs very little on the server side and buys you the full migration window."
* "At this point, clients who cannot upgrade on your schedule are still fully functional."

---

### Slide 15 — Transition Phase

**Slide Title:** Phase 2: Transition
**Subtitle / key message:** *Migrate consumers. Measure. Communicate.*

**Slide contents:**
* Update clients to read the new field (`account_id`)
* Communicate the deprecation of the old field (`user_id`)
  * API documentation update
  * Deprecation header in responses (`Deprecation:`, `Sunset:`)
  * Direct notification for known consumers
* **Measure adoption — this is the most critical step:**
  * Track per-consumer, per-field usage via observability tooling
  * Define a migration-complete signal: what does zero `user_id` reads look like in logs/metrics?
  * Without this signal, the Contract phase cannot happen safely
* Support mixed-version clients for a defined deprecation window

**Speaker notes:**
* "The transition phase is where the operational discipline matters most."
* "Deprecation without measurement is a guess. You need signal: 'is any client still reading user_id?'"
* "The deprecation window is a commitment: you're promising consumers they have at least N days to migrate."
* "In B2B SaaS, this window is often negotiated with specific customers."
* "And the hardest part isn't the communication — it's the measurement. We'll look at why on the next slide."

---

### Slide 16 — Why Measurement Is the Hardest Part

**Slide Title:** Without Measurement, You Never Get to Contract
**Subtitle / key message:** *No signal → no courage to delete → the API becomes a museum.*

**Slide contents:**
* No measurement → no confidence to delete
* No deletion → old fields accumulate
* Cluttered API → new users can't tell current from legacy
* **Key constraint: the server cannot see which field a client reads**
  * You send both fields; the client silently picks one — no wire-level signal
  * So monitoring must rely on server-side inference, not client cooperation
* **Without measurement, you always get Expand — but never Contract**

**Speaker notes:**
* "Here's the mental model to drive home: monitoring is not just a nice-to-have — it's what gives the team the courage to complete Phase 3."
* "Important clarification: the server cannot directly observe which JSON field a client reads. You send both; the client picks one silently."
* "So the monitoring playbook is server-side inference — not asking clients to report back. That would be contradictory: if you could force clients to add a header, you could force them to use the new field."
* "Without a clear zero-read signal, engineers are afraid to delete. So they don't. And the old field stays."
* "Over time, the API becomes a museum of past decisions. New users reading the schema don't know which fields are current and which are kept 'just in case.'"
* "The question to ask your team: if you wanted to remove a field today, what would prove it was safe? If there's no answer, that's the gap."

---

### Slide 17 — Server-Side Signals for Migration Confirmation

**Slide Title:** Practical Monitoring: No Client Change Required
**Subtitle / key message:** *Infer migration from what you already have — version, behavior, and controlled experiments.*

**Slide contents:**
* **Version inference:** map `client_version` (User-Agent / API key / OAuth client ID) to known field usage
* **Behavioral inference:** if subsequent requests use `account_id` as a parameter, the client has migrated
* **Canary omission:** briefly stop sending `user_id` to a small slice of known-migrated traffic; no errors = confirmed
* **Consumer registry:** track who calls you, what version they run, and their declared migration status
* **Gate rule:** zero legacy-field dependency signals from any production consumer for 14 consecutive days

**Speaker notes:**
* "These four signals require zero client-side changes. You're using information already present in production traffic."
* "The most common signal is client version. If you know app v3.2+ reads `account_id`, track traffic by version. When v3.1 and below hit zero, migration is confirmed."
* "Behavioral inference works when the new field also appears in requests — not always available, but strong when it is."
* "Canary omission is your final-proof tool: briefly omit the old field for a small slice. If no errors spike, the client has genuinely moved. Do this with rollback guardrails."
* "The consumer registry is organizational, not technical: know your consumers, track their versions, maintain contact for outreach."
* "The gate rule ties it together: define a threshold (e.g., 14 days of zero signals) before you proceed to Contract phase."

---

### Slide 18 — Contract Phase

**Slide Title:** Phase 3: Contract
**Subtitle / key message:** *Remove only after evidence — not after a calendar date.*

**Slide contents:**
* Remove the old field (`user_id`) from the server response
* Preconditions (all must be met):
  * Version inference shows zero traffic from legacy-dependent versions for ≥ 14 days
  * Canary omission test passed without errors for the remaining consumers
  * Deprecation window has elapsed
  * Known consumers confirmed migrated via registry or direct sign-off
* Final step: deploy removal; monitor error rates for 24h rollback window

**Speaker notes:**
* "This slide connects directly to the signals from the previous slide: version inference, canary omission, and consumer registry are the three inputs to the removal decision."
* "The most common mistake at this phase: removing on a calendar date without evidence. A date says 'you had time.' Evidence says 'you have moved.' These are different."
* "The preconditions are not a checklist to rush through — they are gates. If any one is not met, you stay in Transition."
* "The 24h rollback window is your safety net: if a consumer you didn't know about breaks, you can re-add the field within one deploy cycle."
* "If you can't satisfy these preconditions today, that's not a blocker to starting Expand & Contract — it's a signal to invest in the observability that makes Contract possible later."

---

### Slide 19 — Walkthrough: `user_id` → `account_id`

**Slide Title:** Walkthrough: Rename Without Breaking
**Subtitle / key message:** *Three deployments. Zero client outages.*

**Slide contents:**

| Phase | Server produces | Old client | New client | Action |
|---|---|---|---|---|
| Before | `user_id` | reads `user_id` ✓ | — | — |
| Expand | `user_id` + `account_id` | reads `user_id` ✓ | reads `account_id` ✓ | Deploy server |
| Transition | `user_id` + `account_id` | migrates → `account_id` ✓ | reads `account_id` ✓ | Update + monitor clients |
| Contract | `account_id` | *(migrated)* ✓ | reads `account_id` ✓ | Remove `user_id` from server |

* Logs and metrics show: when does `user_id` read count reach zero?
* That zero-read signal is the gate for Phase 3

**Speaker notes:**
* "Walk through the table row by row — it makes the three-phase concept concrete."
* "Notice: the server makes two independent deployments. Each is backward-compatible at the time it ships."
* "The gate for Phase 3 is an observability question, not a calendar question."

---

### Slide 20 — Divider: When to Break the Contract Intentionally

---

### Slide 21 — Real-World Precedents

**Slide Title:** When Compatibility Is Not Enough
**Subtitle / key message:** *Some changes cannot be made backward-compatible without compromising security, compliance, or long-term platform integrity.*

**Slide contents:**
* **Google OAuth security tightening (2017–2022)**
  * Short-term: all OAuth consumers required migration
  * Long-term: security posture, ecosystem health, elimination of insecure grant types
* **LINE API platform overhaul (September 2016)**
  * BOT API replaced entirely by Messaging API v2
  * Short-term: forced full migration for all developers
  * Long-term: scalability, feature completeness, platform renewal
* Pattern in both cases:
  * Technically rational decision with a long deprecation window
  * Migration path was provided; the change was not arbitrary

**Speaker notes:**
* "These are two well-known examples where the breaking change was the right call — not a failure."
* "Both decisions caused short-term disruption and were still worth making."
* "The lesson is not 'go ahead and break things.' It's: a breaking change can be a responsible, governed decision."
* "Ask: what would have happened to Google's security posture if they kept backward-compatible support for deprecated OAuth flows indefinitely?"

---

### Slide 22 — Criteria for Justified Breaking Change

**Slide Title:** When a Breaking Change Is Justified
**Subtitle / key message:** *Breaking change is a last resort — not a design style.*

**Slide contents:**
* **Use breaking change only if all are true:**
  * Additive path is not viable (security/compliance/platform integrity)
  * Migration path + deprecation window are defined
  * Observability can confirm migration and safe removal
* **Decision rule:** breaking change is a governed exception, not a convenience
* **B2B reality:** influential customer pressure is a risk input, not a veto

**Speaker notes:**
* "The criteria slide is the complement to the rest of this session."
* "Most of today is about avoiding breaking changes. This slide says: if you have to make one, here is what makes it a responsible decision."
* "In B2B SaaS, you will meet the customer who says 'we cannot migrate, not now, not ever.' That's a real conversation. But 'our biggest customer hasn't migrated' is not a reason to hold back a security-required change indefinitely."
* "Framing to leave in learners' minds: compatibility is the discipline; breaking is the governed exception."

---

### Slide 23 — Divider: API Styles & Evolution Rules

---

### Slide 24 — Bridge: Same Goal, Different Contracts

**Slide Title:** Same Change, Different Compatibility Styles
**Subtitle / key message:** *Compatibility depends on where the contract lives.*

**Slide contents:**
* We are solving the same problem in all styles: evolve without breaking consumers
* But the contract location is different:
  * REST: behavior + docs
  * gRPC: `.proto`
  * GraphQL: schema
* Therefore, "safe change" and "breaking change" are judged differently by style

**Speaker notes:**
* "Transition point: the goal is constant across stacks, but the safety rules are not."
* "Before diving into each style, anchor this mental model: where the contract lives determines what you can change safely."
* "In REST, behavior drift is easy to miss; in gRPC, field-number discipline is central; in GraphQL, deprecate-first workflow is the default."
* "So the next three slides are not three APIs to memorize - they are three contract locations with different compatibility guardrails."

---

### Slide 25 — REST: Where the Contract Lives

**Slide Title:** REST — The Implicit Contract
**Subtitle / key message:** *The contract lives in behavior and docs. That gap is where most breakage hides.*

**Slide contents:**
* Contract lives in behavior + docs (OpenAPI)
* Protocol does not enforce compatibility rules
* Safe default: additive fields; risky default: rename/remove
* Highest behavioral-coupling risk: clients can easily depend on undocumented status-code, nullability, and default-value conventions
* Concrete contract sample (OpenAPI):

```yaml
paths:
  /api/profile:
    get:
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  user_id:
                    type: string
                    deprecated: true
                  account_id:
                    type: string
                  name:
                    type: string
```

**Speaker notes:**
* "REST's strength is simplicity and ubiquity. Its weakness for evolution is that the contract is implicit."
* "When there is no schema enforcement at the wire level, clients fill the gap with assumptions."
* "A client that has been reading a field as always-non-null has created an implicit contract. You didn't document that; they still depend on it."
* "This is why REST is where behavioral coupling is easiest to miss: the client may be coupled to your runtime semantics long before anyone writes that dependency down."
* "The discipline REST requires is: treat your OpenAPI spec as the real contract, validate it in CI, and track consumer behavior in production."

---

### Slide 26 — gRPC: Where the Contract Lives

**Slide Title:** gRPC — The Explicit Contract
**Subtitle / key message:** *The `.proto` file is the source of truth. Evolution rules are encoded in the format.*

**Slide contents:**
* Contract lives in `.proto` files (shared, versioned source of truth)
* Field numbers are identity; names are labels
* Safe default: add new field numbers; never reuse retired numbers
* Mark removed numbers/names as `reserved`
* Concrete contract sample (`.proto`):

```proto
message UserProfile {
  reserved 1;
  reserved "user_id";

  string account_id = 2;
  string name = 3;
}
```

**Speaker notes:**
* "In Protobuf, when data is serialized, field names are not on the wire — only field numbers are."
* "That means renaming a field in the .proto file is safe as long as the field number stays the same. The client never sees the name change."
* "The dangerous operation is reusing a retired field number. If field 3 used to be a string and you add field 3 back as an int, clients holding old data will decode garbage."
* "The `reserved` keyword exists precisely to prevent that accident. Use it every time you remove a field."
* "Analogy for non-gRPC readers: REST is like speaking English with names; if we change a word, people can get confused. gRPC is like using a shared codebook with numbers; as long as both sides use the same codebook, we can change internal implementation without misunderstanding."

---

### Slide 27 — GraphQL: Where the Contract Lives

**Slide Title:** GraphQL — The Schema as Contract
**Subtitle / key message:** *Deprecate first. Delete later. Let the tooling warn clients.*

**Slide contents:**
* Contract lives in the schema (introspectable, tool-visible)
* Safe default: add fields; risky default: remove or change types
* Deprecate first with `@deprecated`, then remove after usage drops
* Tooling gives early warnings in IDE/codegen
* Concrete contract sample (schema + query):

```graphql
type UserProfile {
  user_id: ID @deprecated(reason: "Use account_id")
  account_id: ID!
  name: String!
}

query GetProfile {
  profile {
    account_id
    name
  }
}
```

**Speaker notes:**
* "GraphQL's key evolution advantage is the `@deprecated` directive — it makes the intent visible in tooling before the field disappears."
* "In REST, deprecation is usually a documentation note or a response header. In GraphQL, it surfaces directly in the IDE of every client developer."
* "The pattern: mark deprecated, let consumers clean up at their pace (with tooling nudges), then remove after usage drops to zero."
* "Same principle as Expand & Contract — just with the schema as the instrument and `@deprecated` as the transition signal."

---

### Slide 28 — Side-by-Side: Where the Contract Lives

**Slide Title:** Comparison: Contract Location & Evolution Rules
**Subtitle / key message:** *Compatibility looks different because the contract lives in a different place.*

**Slide contents:**

| | REST | gRPC | GraphQL |
|---|---|---|---|
| Contract location | Behavior + docs (OpenAPI) | `.proto` file | Schema |
| Enforcement | None at protocol level | Field numbers + reserved | Type system + `@deprecated` |
| Easiest safe change | Add optional field | Add field with new number | Add field or type |
| Most dangerous trap | Undocumented behavior assumption | Reusing a retired field number | Silent field removal |
| Deprecation signal | Doc / response header | Comment + `reserved` | `@deprecated` directive (IDE-visible) |

* **Discussion prompt:** For your current stack, where is your real contract written today, and what makes compatibility failures most likely?

**Speaker notes:**
* "This table is the handout-ready summary. Each row is a real decision point."
* "The discussion prompt is important: ask learners to name their stack and map the table to their own situation."
* "The goal is not to pick a winner — it's to know where your contract actually lives so you know what discipline to apply."

---

### Slide 29 — Exercise 1: Rename a Critical Field Safely

**Slide Title:** Exercise 1 (15 min): Design a Safe Field Rename
**Subtitle / key message:** *Apply Expand & Contract to a real scenario.*

**Slide contents:**
* **Scenario:** A widely-used API response contains `user_id`. The domain now requires `account_id`.
  * Consumers include: a web frontend, two mobile app versions, a partner integration, and an internal analytics pipeline.
* **Design tasks:**
  1. Draw the response shape in each phase (Expand, Transition, Contract)
  2. How long do both fields stay available? What determines the window?
  3. What telemetry do you need to confirm consumers have migrated?
  4. What is the exact rule for when `user_id` can be removed?

**Speaker notes:**
* "Give participants 12–13 minutes to work individually or in pairs, then debrief together."
* "The scenario is intentionally varied: web clients, mobile, partner, and internal pipeline represent four different upgrade velocities."
* "Most groups will struggle with question 4 — that's intentional. The correct answer is evidence-based, not date-based."

---

### Slide — Exercise 1: Debrief

**Slide Title:** Exercise 1 — Debrief
**Slide contents:**
* Which client type was hardest to migrate, and why?
  * Expected: mobile app old versions — outside your release control; user install rate determines the window
* What would make the plan unsafe?
  * No field-level observability (you can't measure zero-reads)
  * No deprecation window commitment to the partner
  * Removing on a date without confirmation from the analytics pipeline team
* What telemetry proves readiness?
  * Zero reads of `user_id` in access logs, by consumer/client version, for a defined period

**Speaker notes:**
* "Walk through each debrief point; draw out the group's answers before giving the model answer."
* "The mobile client point is the most important: if you have 2-year-old app versions in the field, your deprecation window must account for that reality."
* "Connect back to the framing rule: your server can deploy today; your clients might not."

---

### Slide 30 — Divider: Versioning Strategies + Rollout Safety

---

### Slide 31 — Versioning Strategies

**Slide Title:** Versioning Strategies: Three Approaches
**Subtitle / key message:** *Version numbers help routing and governance — they do not replace compatibility discipline.*

**Slide contents:**
* **URL path versioning** (`/v1/users`, `/v2/users`)
  * Simple to understand, explicit routing, easy to monitor per-version traffic
  * Creates parallel maintenance: `/v1` and `/v2` must both be kept running until migration completes
* **Header or media-type versioning** (`Accept: application/vnd.api+json;version=2`)
  * Cleaner URL space; harder to test and observe
  * Less common outside strict REST API design
* **Schema-first evolution without explicit version bumps**
  * Rely on additive discipline and deprecation; avoid frequent new versions
  * Common in gRPC and GraphQL; fits teams with strong schema governance

**Speaker notes:**
* "The key distinction: versioning is a routing and governance tool. It does not prevent breakage — that's what additive discipline and deprecation windows are for."
* "URL versioning is the most common because it's the most visible. Traffic by version is easy to observe."
* "The danger of URL versioning done poorly: teams ship `/v2` and never sunset `/v1`. Now you're maintaining two APIs indefinitely."

---

### Slide 32 — Versioning Tradeoffs

**Slide Title:** Versioning Tradeoffs
**Subtitle / key message:** *More versions = more clarity and more maintenance cost.*

**Slide contents:**
* Explicit versions:
  * Pros: clear consumer isolation, easy A/B observability, unambiguous rollback
  * Cons: parallel maintenance burden; each version is a separate compatibility contract to uphold
* Too many versions can hide weak compatibility discipline
* Not every change deserves a new version:
  * Additive changes and semantic-neutral refactors: no new version needed
  * Reserve new versions for genuinely incompatible changes with a migration plan
* Rule: a version without a sunset date is a forever commitment

**Speaker notes:**
* "The versioning hygiene principle: every `/v2` should come with a committed date for `/v1` sunset."
* "Teams that version too eagerly often end up supporting v1, v2, v3, v4 in parallel — each representing a different era of technical debt."
* "Versioning is useful, but additive discipline is what actually prevents outages. A well-designed v1 that evolves additively is better than four poorly-maintained versions."

---

### Slide 33 — Rollout Safety Tools

**Slide Title:** Rollout Safety: Proving a Change Is Safe
**Subtitle / key message:** *Deprecation windows, observability, and feature flags are operational tools — not formalities.*

**Slide contents:**
* **Deprecation policy and sunset communication**
  * Commit to a window; publish it; enforce it
  * Direct communication to known consumers (especially in B2B)
* **Consumer usage observability**
  * Track per-consumer, per-version, per-field usage in production
  * Zero-read signal is the gate for removal, not a date
* **Feature flags / selective rollout**
  * Expose new response behavior to a subset of traffic first
  * Validate new behavior in production before broad rollout
  * Limit blast radius: if new behavior is wrong, roll back without touching all consumers

**Speaker notes:**
* "These three tools form the operational layer of API evolution. The Expand & Contract pattern tells you what to do; these tools give you the signal to know when it's safe."
* "Deprecation without observability is a guess. You need to measure 'is anyone still reading `user_id`?'"
* "Feature flags let you introduce a new behavior additively — expose it to 10% of traffic, confirm it's correct, then graduate to 100%."
* "Connect to Session 2: observability as a production tool for proving safety is the same discipline applied here."

---

### Slide 34 — Exercise 2: Feature Flag Rollout

**Slide Title:** Exercise 2 (10 min): Gradual Rollout with a Feature Flag
**Subtitle / key message:** *How do you introduce a new response behavior without risking all consumers at once?*

**Slide contents:**
* **Scenario:** You are changing how `order_status` is computed — same field name, new logic.
  * The change is backward-compatible by definition, but the new logic is complex.
  * You want to expose it to 10% of traffic first.
* **Design tasks:**
  1. What does the feature flag gate — field value, response shape, or both?
  2. What is your success signal? What does "working correctly" look like in metrics/logs?
  3. What is your rollback signal? At what threshold do you roll back?
  4. When do you graduate to 100%?

**Speaker notes:**
* "This exercise is intentionally short — the concepts are already in place, this is applying them."
* "The key question is question 2: many teams define rollout success as 'no new errors.' That's necessary but not sufficient. The right signal is 'new behavior produces the expected business outcomes.'"
* "Question 3: rollback thresholds should be pre-defined before the rollout, not decided in panic during an incident."

---

### Slide 35 — Module Summary

**Slide Title:** Module Summary
**Subtitle / key message:** *Compatibility-first evolution with evidence-based removal.*

**Slide contents:**
* API evolution is a compatibility problem, not just a code-change problem
* Expand & Contract is the default migration playbook
* Measurement is mandatory: without migration signals, Contract phase cannot close
* Behavioral coupling matters: changing semantics, defaults, or error handling is still a contract change
* Contract location changes by style: REST (behavior/docs), gRPC (`.proto`), GraphQL (schema)
* Breaking change is a governed exception, not a convenience

**Speaker notes:**
* "This summary slide is the bridge between exercises and final takeaway."
* "Reinforce the sequence: compatibility mindset, migration pattern, observability discipline, then style-specific rules."
* "If learners remember one mental model, make it this: expand safely, measure honestly, contract only with evidence."
* "The next slide turns this into three concise takeaways they can apply immediately."

---

### Slide 36 — Key Takeaways

**Slide Title:** Wrap: 3 Things to Remember
**Subtitle / key message:** *Compatibility is a runtime concern, not just a schema concern.*

**Slide contents:**
* **1. Always expand before you contract**
  * A field rename is a three-phase operation, not a one-line diff
* **2. Compatibility is a runtime concern, not just a schema concern**
  * Clients depend on behavior, not only on documented contracts
  * Behavioral coupling is what turns small semantic tweaks into real outages
  * Measure migration; remove only after evidence
* **3. Versioning helps — but compatibility discipline prevents outages**
  * Version numbers manage routing and governance
  * Additive discipline and deprecation policy are what keep clients running

**Speaker notes:**
* "These three takeaways map directly to the three most common mistakes: renaming in one step, trusting the schema instead of measuring behavior, and shipping a new version instead of evolving the existing one carefully."
* "Leave them with this question: 'In your current codebase, which of these three is the weakest link?'"
* "Thank them for their time and point to the handout for the compatibility checklist and the Expand & Contract playbook."

---

### Slide 37 — What's Next

**Slide Title:** What's Next: 101-06 Deployment Safety
**Subtitle / key message:** *Ship risky changes safely with controlled rollout and fast rollback.*

**Slide contents:**
* Next module: **Microservices 101-06 — Deployment Safety**
* Focus areas:
  * Canary and blue/green release strategies
  * Feature flags as runtime safety controls
  * Rollback criteria, blast-radius control, and release guardrails
* Connection to this module:
  * API evolution defines *what* to change safely
  * Deployment safety defines *how* to ship those changes safely

**Speaker notes:**
* "Session 5 taught how to evolve contracts without breaking consumers; Session 6 teaches how to deploy those changes with minimum production risk."
* "The mental handoff is simple: this module gave you compatibility rules, the next module gives you release mechanics."
* "If a change is contract-safe but rollout-unsafe, you can still cause incidents. We need both."
* "Invite learners to bring one real change from their system to map into canary, feature-flag, and rollback plans in 101-06."

---
