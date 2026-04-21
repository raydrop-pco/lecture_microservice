# Microservices 101: Series Overview
**Foundations for Distributed Systems Engineering**

*By Ray Suginaka*

## Welcome to Microservices 101

**Target Audience:** Engineers and developers who are new to microservices design and distributed systems.
**Primary Goal:** To reshape your architectural mindset. We focus deeply on the "Why" behind microservice patterns rather than being an exhaustive manual of the "How." Mastering these underlying mechanisms and learning how to apply them dynamically is far more important than simply memorizing step-by-step instructions.

**Welcome to Microservices 101!** 

Transitioning from a monolith or modular-monolith world into the microservices ecosystem is an exciting engineering journey for application developers. However, thriving in this world requires far more than just adopting new technical tools—it requires a positive and fundamental shift in your **mental models**. Without understanding this mental model, you cannot design good systems even with AI.

In traditional architectures, you might rely on safe, single-database transactions and instantaneous function calls. In distributed systems, networks will fail, databases are isolated, and consistency is eventual. 

Rather than overwhelming you with a comprehensive list of implementation details, this series is explicitly designed to be **practical and punchy**. We aim to equip you with the foundational principles—the core reasons *why* distributed systems are built the way they are—so you can make confident, resilient design decisions.

## Learning Paths

* **The 101 Series** is the foundational baseline. It is highly recommended for any engineer joining a microservices development team to help them build their core architectural basics.
* **The 201 Series** is the advanced path. It is targeted at Tech Leads and Senior Engineers who are responsible for designing systems from scratch and making high-stakes architectural decisions. (Coming soon)

## Microservices 101: The Foundation

| Session | Title | Abstract |
| :--- | :--- | :--- |
| **[101-01]** | **Idempotency & Consistency** | Staying correct under retries, duplicates, and lag. |
| **[101-02]** | **Resilience & Observability** | Containing failure and reducing MTTR in distributed calls. |
| **[101-03]** | **Data Boundaries & Ownership** | Own the domain and move data safely. |
| **[101-04]** | **Workflows & Messaging** | Moving from jobs/batch to event-driven chains safely. |
| **[101-05]** | **API Evolution** | Evolve contracts without breaking consumers (expand/contract + compatibility rules). |
| **[101-06]** | **Deployment Safety** | Shipping changes safely with canary/blue-green, flags, and rollbacks. |

### Module Deep Dives

#### 101-01: Idempotency & Consistency
**How to stay consistent when retrying.**
The series starts with correctness. If a request times out and the client retries, how do you prevent double charges? This module establishes the foundations of Idempotency Keys and how to expose temporary "in-progress" states for eventual consistency.

#### 101-02: Resilience & Observability
**How to fail gracefully and debug fast.**
Even with correct logical design, systems fail. This module covers stability patterns like bounded retries, exponential backoff, jitter, and circuit breaking/timeouts. It also introduces observability practices so you can diagnose root causes quickly when things break.

#### 101-03: Data Boundaries & Ownership
**How to live without SQL Joins.**
The "Shared Database" is a trap—one small schema change in Service A breaks Service B. This module covers Schema Ownership, why services should only talk to their own databases, and how to use the Outbox Pattern to avoid dual-write failures. 

#### 101-04: Workflows & Messaging
**Moving from Batch Jobs to Event Chains.**
We explore when to move from waiting for immediate responses to queuing asynchronous tasks. We contrast Orchestration with Choreography, and explore the Saga Pattern for triggering compensating actions (the "Undo" button) when distributed processes fail.

#### 101-05: API Evolution
**How to Change Without Breaking the World.**
If you update a critical field name, the mobile app might crash for fifty thousand users. This module discusses API styles (REST, gRPC, GraphQL), API versioning strategies, and the robust "Expand & Contract" deployment pattern so you never break existing clients.

#### 101-06: Deployment Safety
**Decoupling deployments from releases.**
"It worked in staging..." — why production deployments are inherently risky. We put it all into production safely by decoupling code shipping from feature enablement. We cover Zero-Downtime deployments (Blue-Green) and Blast Radius Reduction (Canary Releases and Feature Flags). 

---

## Beyond 101 (Coming Soon)
After completing the **101 Foundation Series**, an advanced **201 Series** will be waiting for you. The 201 series is designed for Tech Leads of development teams and Senior Engineers who will design systems from scratch and make high-stakes architectural decisions.
