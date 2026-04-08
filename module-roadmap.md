# Full Roadmap

## Target Audience:

Engineers and developers who are new to microservices design and distributed systems.

## Primary Goal:

To reshape your architectural mindset. We focus deeply on the "Why" behind microservice patterns rather than being an exhaustive manual of the "How."

## Microservices 101: The Foundation (Core Survival)

| Session | Title                       | Big Idea                                                                             |
| ------- | --------------------------- | ------------------------------------------------------------------------------------ |
| 101-01  | Idempotency & Consistency   | Staying correct under retries, duplicates, and lag.                                  |
| 101-02  | Resilience & Observability  | Containing failure and reducing MTTR in distributed calls.                           |
| 101-03  | Data Ownership & Boundaries | Living without shared schemas and cross-service joins.                               |
| 101-04  | Workflows & Messaging       | Moving from jobs/batch to event-driven chains safely.                                |
| 101-05  | API Design for Evolution    | Evolve contracts without breaking consumers (expand/contract + compatibility rules). |
| 101-06  | Deployment Safety           | Shipping changes safely with canary/blue-green, flags, and rollbacks.                |

## Microservices 201: Mastery & Scale (Advanced Patterns)

| Session | Title                                 | Big Idea                                                                                    |
| ------- | ------------------------------------- | ------------------------------------------------------------------------------------------- |
| 201-01  | Advanced Data (CQRS & Event Sourcing) | Complex read/write requirements at scale.                                                   |
| 201-02  | Security & Multi-Tenancy              | Isolation, identity, least privilege, and noisy-neighbor control at SaaS scale.             |
| 201-03  | High Performance & Scale              | Latency budgets, saturation, caching, capacity under real traffic.                          |
| 201-04  | Cloud Leverage (Buy vs Build)         | Managed services choices that accelerate delivery responsibly.                              |
| 201-05  | Contracts & Confidence (Testing)      | Turning boundaries into automated safety gates (compatibility, invariants, mixed versions). |


# Microservices 101: The Foundation (Core Survival)

### 101-01: Idempotency & Consistency
*How to stay correct when retrying.*
* **Intro:** The "Network is Unreliable" reality.
* **Concept A:** Idempotency 101 (Keys and Deduplication).
* **Ex 1 (Design):** Designing a safe "Order Placement" API.
* **Concept B:** Eventual Consistency 101 (Why things lag).
* **Ex 2 (Scenario):** The "Balance hasn't updated yet" customer support simulation.
* **Wrap:** Rules of thumb for safe retries.

### 101-02: Resilience & Observability
*How to fail gracefully and debug fast.*
* **Intro:** The "Cascading Failure" nightmare.
* **Concept A:** Resilience Starter Kit (Timeouts, Breakers, Bulkheads).
* **Ex 1 (Design):** Protecting a service from a slow downstream dependency.
* **Concept B:** Observability (Logs vs. Metrics vs. Traces).
* **Ex 2 (Debugging Game):** Finding the "needle in the haystack" using distributed traces.
* **Wrap:** MTTR (Mean Time to Recovery) as the key metric.

### 101-03: Data Boundaries & Ownership
*How to live without SQL Joins.*
* **Intro:** Why "Shared Truth" (Shared DB) is a scaling handcuff.
* **Concept A:** Schema Ownership & Reference by ID.
* **Ex 1 (Design):** Splitting a 60-column "Master Product" table into four bounded contexts.
* **Concept B:** The Outbox Pattern (Avoiding the "Dual-Write" failure).
* **Ex 2 (Scenario):** Designing a reliable price-update propagation flow.
* **Wrap:** Share IDs, not tables.

### 101-04: Workflows & Messaging
*Moving from "Batch Jobs" to "Event Chains."*
* **Intro:** Evolution of sequences: Job Execution → Queues → Event-Driven.
* **Concept A:** Orchestration (The Conductor) vs. Choreography (The Dance).
* **Ex 1 (Mapping):** Designing a "Refund Process" using both styles.
* **Concept B:** The Saga Pattern & Compensating Actions (The "Undo" button).
* **Ex 2 (Failure Simulation):** Payment succeeded but Inventory failed—design the "Undo" event flow.
* **Wrap:** When to use a central brain vs. a decoupled dance.

### 101-05: API Evolution
*How to Change Without Breaking the World.*
* **Intro:** You updated a field name from `user_id` to `account_id`, and suddenly the mobile app crashes for 50,000 people.
* **Concept A:** Expansion & Contraction Pattern. How to safely migrate fields.
* **Concept B:** API styles & contracts — REST vs gRPC vs GraphQL
* **Concept C:** Versioning Strategies (Headers vs. URL paths).
* **Ex 1:** Renaming a critical API field with zero downtime.
* **Concept D:** Backward Compatibility (Why you should never "break the contract").
* **Ex 2:** Using a Feature Flag to safely roll out a breaking change to 10% of users.
* **Wrap:** Always expand before you contract; never break existing clients.

### 101-06: Deployment Safety
*Shipping changes safely with canary/blue-green, flags, and rollbacks.*
* **Intro:** "It worked in staging" — why production deployments are inherently risky.
* **Concept A:** Zero-Downtime Deployments (Blue-Green vs. Rolling updates).
* **Ex 1:** Diagnosing a connection drain failure during a Rolling update.
* **Concept B:** Blast Radius Reduction (Canary Releases & Feature Flags).
* **Ex 2:** Using a Feature Flag to instantly roll back a broken feature without a code deploy.
* **Wrap:** Decouple your releases (shipping code) from your deployments (enabling features).


---

# Microservices 201: Mastery & Scale (Advanced Patterns)

| Session | Title | The "Big Idea" |
| :--- | :--- | :--- |
| **201-01** | Advanced Data (CQRS & Event Sourcing) | Handling complex read/write requirements. |
| **201-02** | Security & Multi-Tenancy | Protecting data in a SaaS world. |
| **201-03** | High Performance & Scale | Optimizing for high-traffic environments. |
| **201-04** | Cloud Leverage (Buy vs. Build) | Choosing managed services to accelerate delivery. |
| **201-05** | Contracts & Confidence (Testing) | Turning boundaries into automated safety gates. |

### 201-01: Advanced Data (CQRS & Event Sourcing)
*Handling complex read/write requirements.*
* **Concept A:** CQRS (Separating the "Change" from the "View").
* **Ex 1:** Designing a read-optimized dashboard for a high-write system.
* **Concept B:** Event Sourcing (The Log is the Truth).
* **Ex 2:** Reconstructing state from an event log to audit a suspicious transaction.

### 201-02: Security & Multi-Tenancy
*Protecting data in a SaaS world.*
* **Concept A:** Identity & Service-to-Service Auth (JWT/mTLS).
* **Ex 1:** Threat-modeling a service boundary.
* **Concept B:** Data Isolation Patterns (Row-Level Security & Valet Key).
* **Ex 2:** Designing a secure file-upload flow that offloads to cloud storage.

### 201-03: High Performance & Scale
*Optimizing for high-traffic Python/Cloud environments.*
* **Concept A:** Async Programming & Event Loops (Handling blocking calls).
* **Ex 1:** Identifying bottlenecks in an asynchronous microservice.
* **Concept B:** Caching Strategies & Latency Budgeting.
* **Ex 2:** Tuning a call-chain to meet a <200ms P99 requirement.

### 201-04: Cloud Leverage (Buy vs. Build)
*Choosing managed services to accelerate delivery.*
* **Concept A:** Evaluating Managed Workflows vs. Custom Infrastructure.
* **Ex 1:** Replacing a custom "State Machine" with a cloud-native workflow engine.
* **Concept B:** Infrastructure as Code (IaC) & Deployment Automation.
* **Ex 2:** Designing a "One-Click" environment teardown and rebuild.

### 201-05: Contracts & Confidence (Testing) 
*Turning boundaries into automated safety gates (compatibility, invariants, mixed versions).*
* **Intro:** Moving beyond fragile end-to-end integration tests.
* **Concept A:** Consumer-Driven Contract Testing.
* **Ex 1:** Writing a test to guarantee the Order Service never breaks the Payment Service's schema expectations.
* **Concept B:** Chaos Engineering & Invariants.
* **Ex 2:** Running a simulated network partition to verify the Outbox pattern correctly resumes processing.
* **Wrap:** Confidence comes from testing the boundaries, not just the logic.

---

### Delivery Strategy
* **The 101 Series** is the "Entry Permit." Every new engineer should complete this within their first 90 days.
* **The 201 Series** is for "Tech Leads" and "Senior Path" engineers who are designing new systems from scratch.

