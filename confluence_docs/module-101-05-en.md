# Microservices 101-05: API Design for Evolution

Evolve Contracts Without Breaking Consumers

## 1. Introduction

In Module 1, we learned how to make individual operations safe under failure through idempotency and eventual consistency. In Module 2, we designed for failure and built observability to find problems quickly. Module 3 addressed data boundaries and ownership. Module 4 moved up to the application layer: communication models and workflow coordination patterns. Module 5 focuses on the **contract layer**.

Module 4 introduced **behavioral coupling**: a client can become dependent not just on another service being available, but on its specific runtime behavior, such as response semantics, defaults, and error conventions. Module 5 carries that same perspective into API evolution, where those assumptions become compatibility risks over time.

An **API contract** is a formal, shared agreement between a service provider and its consumers about how they interact.

It defines:
* What data is exchanged (the schema or structure).
* How to request it (the protocol and method).
* Behavioral expectations, such as data types, required fields, and errors.

Think of it as a "legal document" between services: as long as the provider adheres to the contract, consumers can rely on the service without knowing its internal implementation details.

After APIs run in production for a while, a new category of risk emerges: **API evolution**. Fields are renamed, endpoints are replaced, and behaviors are corrected. In a monolith, you deploy once and every caller sees the update immediately. In a distributed system, the server and its consumers deploy on independent schedules. Some consumers, such as mobile apps, third-party integrations, and embedded devices, may never upgrade on your timeline.

This module teaches you how to evolve published APIs deliberately, with techniques to maintain compatibility during migration and operational tools to verify safety before removing old behavior.

> 💡 The real question is never "did we change the code?" — it's "can old and new clients still work during and after the rollout?"

### The Mental Model for This Module

In the monolithic world, we often insulated consumers from change through lengthy stakeholder negotiations and synchronized release windows, effectively trading speed for safety. In a distributed architecture, that same control model does not scale: coordination overhead grows until delivery stalls. The mindset must shift from "negotiating release timing" to "designing for inevitable coexistence."

This shift is vital because the core value of microservices is independent change and release. When contracts are designed for coexistence, teams can innovate quickly without requiring every API consumer to upgrade in lockstep.

---

## 2. Compatibility 101 — What Actually Breaks Clients

### What Is an API Contract?

The API contract is not just the OpenAPI spec or the Protobuf file. It is **everything the client observes and relies on** — including undocumented behavior. A field that was always non-null in practice creates an implicit contract, even if the spec marks it as nullable. A response that always returned a sorted list becomes a contract, even if sorting was never specified. The API contract reduces behavioral-coupling risk, and it is effective only when the team actively maintains it through compatibility discipline, testing, and observability during evolution.

Compatibility means: **existing clients still work without modification (including implicit rules)**.

### Change Taxonomy

Not all changes are equal. Classifying changes correctly is the first step to managing them safely.

| Category | Description | Risk level |
|---|---|---|
| **Additive changes** | Add a new optional field; add a new endpoint; add a new enum value | Usually safe (but watch enum handling in switch logic) |
| **Semantic changes** | Same field name, different meaning; changed nullability; changed default value; changed error semantics or status-code meaning | **Dangerous** — schema looks unchanged; client logic breaks |
| **Breaking changes** | Remove a field; rename a field; add a required field to a request; change a response shape | Breaks existing clients immediately |

### Common Traps

The most dangerous changes are the ones that *feel* safe:

- **Changing nullability quietly** — a field that was always non-null becomes sometimes-null; client code that never checked for null now crashes
- **Changing response semantics behind the same endpoint** — yesterday `200 {}` meant "not found," today `404` means "not found"; clients may misclassify a valid business case as a system error
- **Reusing an old field name with new meaning** — clients read stale semantics and receive silently corrupted data
- **Assuming all consumers read documentation** — many clients learn from observed behavior, not specs; observed behavior is the real contract
- **Semantic drift behind a stable endpoint** — same URL, same field names, but the meaning of values has shifted

The quick diagnostic: for the `user_id` → `account_id` rename example, which category does it belong to?

**Breaking change** — the old field disappears in a single deploy. Any client still reading `user_id` gets `null` and, if it doesn't guard against null, crashes.

To avoid this kind of breaking change, apply a staged migration pattern instead of a one-step replacement.

---

## 3. Expand & Contract — The Zero-Downtime Migration Pattern

The Expand & Contract pattern is a three-phase migration playbook. It applies any time a change is *breaking-looking* — meaning it would break clients if deployed in a single step.

### The Three Phases

```
Phase 1 — EXPAND          Phase 2 — TRANSITION       Phase 3 — CONTRACT

Server produces:          Server produces:            Server produces:
  user_id  ← old            user_id  ← old              account_id ← new
  account_id ← new          account_id ← new
                                                       Old field removed safely ✓
Old client:  reads        Old client:  migrates
  user_id ✓               → account_id ✓
New client:  reads
  account_id ✓
```

**Phase 1 — Expand:**
- Add the new field or behavior alongside the old one
- Both exist in every response — no client needs to change yet
- This step is safe because it is purely additive

**Phase 2 — Transition:**
- Update consumers to use the new field
- Communicate deprecation via documentation, `Deprecation:` and `Sunset:` response headers, and direct outreach to known consumers
- **Measure migration**: track per-consumer, per-field usage in production
- Define a migration-complete signal: what does zero `user_id` reads look like in logs and metrics?
- Support mixed-version clients for a defined deprecation window

**Phase 3 — Contract:**
- Remove the old field only after evidence that all consumers have migrated
- Treat the deprecation date as a planning signal, not a removal criterion; proceed only when migration data shows readiness.
- Note: Here, "Contract" means shrinking the compatibility surface (retiring legacy fields), not an "API contract" agreement.


### Why Measurement Is the Critical Step

Without measurement, teams always get Phase 1 (Expand) but never reach Phase 3 (Contract). Old fields accumulate, the API becomes a museum of past decisions, and new developers cannot tell which fields are current and which exist "just in case."

A key constraint: **the server cannot directly observe which JSON field a client reads**. Both fields are sent; the client silently picks one. Monitoring must rely on server-side inference, not client cooperation.

**Server-side signals that require no client change:**

| Signal | How it works |
|---|---|
| **Version inference** | Map version identifiers (for example `/v1` vs `/v2`, version header/media type, or client version via User-Agent/API key/OAuth client ID) to known field usage; when legacy-version traffic hits zero, migration is confirmed |
| **Behavioral inference** | If the new field also appears in subsequent requests as a parameter, the client has migrated |
| **Canary omission** | Briefly stop sending the old field to a small slice of known-migrated traffic; zero errors confirm readiness |
| **Consumer registry** | Organizational tracking of who calls you, what version they run, and their declared migration status |

Version inference directly connects to Section 6 (Versioning Strategies + Rollout Safety): whichever versioning strategy you use, version-level traffic is also a migration signal for Contract-phase removal.

Unless you have maintained a consumer registry from the start, client migration status is mostly inferred. Define explicit migration criteria and track them continuously.

**Gate rule:** zero legacy-field dependency signals from any production consumer for ≥ 14 consecutive days before proceeding to Phase 3.

### Walkthrough: `user_id` → `account_id`

| Phase | Server produces | Old client | New client | Action |
|---|---|---|---|---|
| Before | `user_id` | reads `user_id` ✓ | — | — |
| Expand | `user_id` + `account_id` | reads `user_id` ✓ | reads `account_id` ✓ | Deploy server |
| Transition | `user_id` + `account_id` | migrates → `account_id` ✓ | reads `account_id` ✓ | Update + monitor clients |
| Contract | `account_id` | *(migrated)* ✓ | reads `account_id` ✓ | Remove `user_id` from server |

The server makes two independent deployments. Each is backward-compatible at the time it ships. The gate for Phase 3 is an observability question, not a calendar question.

**Preconditions for Phase 3 removal:**
- Version inference shows zero traffic from legacy-dependent client versions for ≥ 14 days
- Canary omission test passed without errors for remaining consumers
- Deprecation window has elapsed
- Known consumers confirmed migrated via registry or direct sign-off
- After removal: monitor error rates for a 24-hour rollback window

---

## 4. When to Break the Contract Intentionally

Most of this module covers how to avoid breaking clients — that discipline is the foundation. But some changes cannot be made backward-compatible without compromising security, compliance, or long-term platform integrity.

**Compatibility is the default — but not a religion.**

### Real-World Precedents

| Case | Short-term cost | Long-term gain |
|---|---|---|
| **Google OAuth security tightening (2017–2022)** | All OAuth consumers required migration | Security posture, ecosystem health, elimination of insecure grant types |
| **LINE API platform overhaul (September 2016)** | Full migration forced for all developers (BOT API → Messaging API v2) | Scalability, feature completeness, platform renewal |

In Google's case, a transition period was provided, but teams delayed migration and ended up making production API changes at the last minute. In LINE's case, the transition window was only about six months, and teams had to rework applications that had just completed testing on BOT API that deprecation was scheduled. In both cases, the change was neither customer-driven nor caused by implementation defects; the hardest discussions were about who should absorb the migration cost and whether the change was truly unavoidable.

One notable point for me was LINE team repeated direct communication with customers to explain why the changes were necessary. Disruptive changes are always painful, but this showed a strong commitment to responsible, well-governed decisions made in the customer's long-term interest.

(But it was still very difficult…)

### Criteria for a Justified Breaking Change

Disruptive changes never be for the developers' design convenience. As a service provider, they should only be justified in the case of governed, exceptional circumstances.

Especially in B2B services worlds, influential customers may resist the transition. While this pressure is important feedback, if the decision is so easily swayed by such claims, there should be other options. "Premium customers haven't migrated yet" is not a reason to indefinitely postpone business-necessary changes. Once a decision has been made to implement a disruptive change, delaying it will usually result in higher costs and risks in the future.

---

## 5. API Styles & Evolution Rules — REST vs. gRPC vs. GraphQL

The goal is the same across all API styles: evolve without breaking consumers. But the **contract location** differs by style, and compatibility rules follow the contract location.

### REST — The Implicit Contract

**Wire format:** Human-readable plain text (usually JSON) over HTTP. 

**Contract sharing:** Usually documented in OpenAPI (Swagger), but HTTP and JSON do not enforce that contract by themselves. The server and client need shared discipline to keep implementation and specification aligned. This flexibility is useful, but it also makes implicit contracts more likely.

**Error Detection:** Breakage often appears only at runtime. For example, if the server starts returning `{ "user_id": 100 }` while a client still expects `{ "user_id": "100" }`, the HTTP transport still succeeds, but the client application may misparse the payload or fail when it tries to use the value.

In REST, the “contract” is often a mix of documentation and observed behavior. That makes REST flexible and widely adopted, but also makes it easy for clients to depend on implicit semantics (status codes, nullability, defaults, pagination quirks) that were never explicitly promised.

**Evolution rules of thumb:**
- Additive fields are usually safe when clients ignore unknown fields
- Rename and removal almost always break someone
- Treat the OpenAPI spec as the authoritative contract; validate it in CI

```yaml
# Example: both fields present during Expand phase
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

### gRPC — The Explicit Contract

**Where the contract lives:** `.proto` files (shared, versioned source of truth)

**Wire format:** Protocol Buffers (binary-serialized data) over HTTP/2. It is compact, machine-oriented, and much less human-readable than JSON. Conceptually, the payload is closer to `(1: 100)` than `{ "user_id": 100 }`: on the wire, compatibility depends on field numbers and wire types, not field names.

**Contract sharing:** A shared `.proto` IDL (Interface Definition Language) file defines message types and RPC methods. Client and server code are generated from that schema, so both sides work from the same source of truth.

**Error Detection::** Errors are often detected before the application layer runs. For example, if the message type is wrong or the client calls an RPC method the server does not expose, gRPC can fail the request before it reaches your business logic.

In gRPC, both sides agree on a strongly typed schema. Importantly, protobuf encodes fields by field number, not field name—so some refactors are safer than in REST.

**Evolution rules of thumb:**
- Never reuse a retired field number — if field 3 was a `string` and you add field 3 back as an `int`, old data decodes as garbage
- Mark removed field numbers and names as `reserved` to prevent accidental reuse
- Additive new fields (new numbers) are usually safe

```proto
// After completing the rename via Expand & Contract:
message UserProfile {
  reserved 1;
  reserved "user_id";       // prevents accidental reuse of field number 1

  string account_id = 2;
  string name = 3;
}
```

### GraphQL — The Schema as Contract

**Wire format:** Plain text, similar to REST. GraphQL query + JSON response over HTTP (text-based request; JSON response).

**Contract sharing:** A strict GraphQL schema (introspectable and tool-visible). The API tells clients exactly what fields, types, and operations are available.

**Where breakage shows up:** Errors are often detected before the application layer runs. If a client asks for a field that is not in the schema, GraphQL can reject the query during validation before your business logic runs.

GraphQL flips the model: clients explicitly request fields, and the server must satisfy those queries. This makes evolution more visible because the schema is central and deprecations can be advertised directly to developers.

**Evolution rules:**
- Mark fields deprecated with `@deprecated` before removing them — do not remove silently
- Add fields freely; remove only after usage drops to zero
- Schema introspection gives continuous visibility into what clients are using

```graphql
type UserProfile {
  user_id: ID @deprecated(reason: "Use account_id")
  account_id: ID!
  name: String!
}
```

This is Expand & Contract — with the schema as the instrument and `@deprecated` as the transition signal.

### Comparison: Contract Location & Evolution Rules

| | REST | gRPC | GraphQL |
|---|---|---|---|
| Contract location | Behavior + docs (OpenAPI) | `.proto` file | Schema |
| Protocol enforcement | None | Field numbers + `reserved` | Type system + `@deprecated` |
| Easiest safe change | Add optional field | Add field with new number | Add field or type |
| Most dangerous trap | Undocumented behavior assumption | Reusing a retired field number | Silent field removal |
| Deprecation signal | Doc / response header | Comment + `reserved` | `@deprecated` directive (IDE-visible) |

**Discussion prompt:** For your current stack, where is your real contract written today? What makes compatibility failures most likely?

---

## 6. Versioning Strategies

API versioning is mostly a **routing and governance choice**: it determines how you run old and new behavior side-by-side and how easily you can see who is using what. It is not, by itself, what prevents outages—compatibility discipline (additive changes, deprecation windows, and usage telemetry) does that.

> *Note* : Use /vN only when you need incompatible behavior to coexist. Don’t put v1.1.1 in URLs—evolve within a major version using Expand & Contract.URLs are routing identifiers, not release notes. Patch/minor versions encourage version sprawl and create a false expectation that clients must track every tiny version. Keep URLs stable and evolve additively; reserve /v2 for genuinely incompatible semantics. If you must distinguish smaller changes, do it via compatibility, not URL churn.

### URL path versioning

The most explicit approach is URL path versioning (for example `/v1/users`, `/v2/users`). This makes the version visible in every log line and dashboard, and it’s easy for consumers to understand. The cost is that you now own parallel maintenance: both versions must stay alive until clients migrate, and if you bump versions too casually you accumulate long-term version sprawl.

### Header or media-type versioning

A cleaner URL space comes from header or media-type versioning (for example `Accept: application/vnd.api+json;version=2`). This keeps endpoints stable while allowing multiple representations or behaviors. In practice, it can be harder to operate: debugging is less transparent, observability and gateway routing often need extra work to segment traffic by version, and consumers may struggle to adopt it without strong tooling.

### Query parameter versioning

Some teams use query-parameter versioning (for example `/users?api-version=2`) as a pragmatic middle ground. It is easy to experiment with and easy to roll out behind gateways, but it can be treated as “optional” and ignored, and you need to be careful with caching layers so that different versions don’t share cached responses.

### Schema-first evolution

Finally, many mature systems—especially gRPC and GraphQL, and increasingly REST with strong governance—prefer schema-first evolution *without frequent explicit version* bumps. The idea is to keep a single “current” interface and evolve it primarily through additive changes and deprecation policy. This avoids running multiple versions in parallel, but it requires discipline: clear compatibility rules, a real deprecation window, and telemetry that proves when old fields or behaviors are no longer used. 


A good default stance for beginners is: prefer additive, schema-first evolution; introduce explicit versions only when you need incompatible behavior to run in parallel or you have long-lived external clients you can’t force to upgrade.

**Versioning hygiene rules:**
- Every new version should come with a committed sunset date for the previous version
- A version without a sunset date is a forever commitment
- Not every change deserves a new version — reserve versions for genuinely incompatible changes with a migration plan
- Too many parallel versions can mask weak compatibility discipline

---

## 7. Rollout Safety

In the Expand & Contract pattern (Section ##3), I introduce *how* and *what* to change, as well as the migration-evidence model in detail: what signals indicate that consumers have moved, and what gate should be met before Contract-phase removal. Here stays at the operational layer: the tooling and rollout controls that make that evidence usable in production.

**Deprecation policy and sunset communication:**
- Publish a clear deprecation policy (how warnings are issued, minimum support window, and what “sunset” means).
- Communicate early and repeatedly: docs, changelog, response headers, and targeted outreach.
- In B2B, treat migrations as a customer success activity: agree on timelines, provide support, and escalate when necessary.

**Consumer usage observability (the removal gate):**
- Instrument consumer traffic by version, field, and client identity in production.
- Use “zero usage” as a gate for removal—but with guardrails:
  - define the observation window (e.g., “zero reads for 30 days”)
  - account for long-tail or seasonal traffic
- Without real usage signals, teams stall in “forever expand,” leaving permanent compatibility debt.

**Feature flags and selective rollout:**
- Expose new response behavior to a subset of traffic first (e.g., 10%)
- Validate new behavior in production before broad rollout
- Define success and rollback signals *before* starting the rollout
- Limit blast radius: if new behavior is incorrect, roll back without affecting all consumers

**Feature flags and selective rollout (for risky behavior changes):**
Expand & Contract handles schema shape well, but behavior/semantics (defaults, sorting, rounding, authorization) can still break consumers even if the schema is unchanged. For these:
- Roll out new behavior to a small cohort first (e.g., 1% → 10% → 50% → 100%).
- Define success and rollback criteria before enabling:
  - error rate / client crash rate
  - conversion or business KPIs (if relevant)
  - latency/timeout changes (new behavior can add load)
- Ensure rollback is fast and safe (ideally server-side toggles, not client releases).


### Notes: Canary release vs feature-flag rollout

A **canary release** is a *deployment-based* rollout. You run **two service versions at the same time** (old + new) and route a small slice of production traffic to the new version (e.g., 1% → 10% → 50% → 100%). If signals look bad, you roll back by routing traffic away from the canary or reverting the deployment.

- **Pros:** validates the *entire* new build in real traffic (code + config + runtime); good for changes that can’t be toggled safely (performance fixes, dependency upgrades, infra changes).
- **Cons:** requires operating **mixed versions** (compatibility concerns, debugging complexity); rollback may still take time; can be noisy if traffic segmentation isn’t clean.

A **feature-flag rollout** is a *behavior-based* rollout. You often need only **one deployed version** (the new code is already shipped), but the new behavior is enabled only for a small cohort of users/tenants/requests. You roll back by flipping the flag off—typically faster than redeploying.

- **Pros:** fast rollback (toggle); supports cohort-based rollout (by tenant, user, region); great for **semantic changes** where schema stays the same (defaults, sorting, rounding, authorization behavior).
- **Cons:** flags add complexity and can become “permanent”; you must test both paths; flags can create hidden combinations (“flag interactions”) if unmanaged.

**Rule of thumb:**  
Use **feature flags** when you can isolate the behavior change behind a toggle and you want fast rollback. Use a **canary release** when the change is tied to a new binary/runtime/infrastructure behavior that can’t be safely switched off at runtime.


### Safe completion checklist (what “done” looks like)
- New contract is live and observed in production
- Consumers have migrated (measured, not assumed)
- Deprecation window completed (or exceptions documented)
- Old contract removed with confidence and monitored for regressions

> Connection to Module 101-02: Observability is the same discipline—measure reality and reduce uncertainty. In 101-02 you measure system health; here you measure client migration and contract usage to make change safe.

---

## 8. Closing: Evolve Deliberately, Remove with Evidence

In a microservices world, we assume change is constant. It’s ideal to keep APIs stable, but sometimes change is unavoidable—security fixes, platform evolution, or simply correcting a domain model. When that happens, the challenge isn’t just updating code. Distributed systems force you to live with mixed versions: servers deploy today, but clients upgrade on their own timelines, and old and new behaviors often coexist longer than anyone expects.

That’s why safe API evolution needs both discipline and evidence. Techniques like **Expand & Contract** (add first, remove later), treating **behavioral coupling** as seriously as schema compatibility, and using **versioning sparingly** for truly incompatible semantics give you a practical playbook. And without **observability**—clear usage signals, per-consumer adoption tracking, and safe rollout tools—you don’t have the confidence to finish the migration. The goal is simple: make change routine, not risky, and evolve contracts without breaking the people who rely on them.


### Bridge: What's Next?

This module addressed *what* to change safely — contract evolution, migration patterns, and observability. Module 6 addresses *how* to ship those changes safely into production.

In **Module 6 (Deployment Safety)**, we cover canary and blue/green release strategies, feature flags as runtime safety controls, and rollback criteria and blast-radius management.

API evolution defines what to change safely. Deployment safety defines how to ship those changes safely. A change can be contract-safe but still cause incidents if the rollout mechanics are wrong — you need both.

---

## Appendix: Production-Ready Checklist

- [ ] All API changes are classified as additive, semantic, or breaking before implementation begins.
- [ ] Breaking-looking changes use the Expand & Contract pattern — no single-step field removals or renames.
- [ ] Server-side monitoring tracks per-consumer, per-field, and per-version usage in production.
- [ ] A migration-complete signal (zero legacy-field reads for ≥ 14 days) is defined before Phase 3 begins.
- [ ] Deprecation is communicated via documentation, response headers (`Deprecation:`, `Sunset:`), and direct consumer outreach.
- [ ] REST: OpenAPI spec is validated in CI and treated as the authoritative contract.
- [ ] gRPC: retired field numbers and names are marked `reserved`; field numbers are never reused.
- [ ] GraphQL: fields are marked `@deprecated` before removal; field usage in production is monitored.
- [ ] Every new API version has a committed sunset date for the previous version.
- [ ] Breaking changes are approved only when the additive path is not viable and a migration path + deprecation window are defined.
- [ ] Rollout uses feature flags or canary deployment to limit blast radius; success and rollback signals are defined before rollout starts.
