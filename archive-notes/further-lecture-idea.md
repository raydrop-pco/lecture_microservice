Proposed by GitHub Copilot

101-01: correctness under retries (idempotency/eventual consistency)
101-02: resilience + observability

---

The most natural next lessons are:

## 101-03: Data Modeling & Storage Patterns for Microservices
* DB-per-service, schema ownership, reference by ID
* Transaction boundaries, outbox/inbox, CDC basics
* Query patterns (API composition, CQRS-lite)
* Exercise: split a monolith table safely without breaking reads

## 101-04: API Design for Evolution
* Backward compatibility, versioning strategies
* Error model, pagination/filtering consistency
* Contract testing and consumer-driven contracts
* Exercise: evolve a live API with zero consumer breakage

## 101-05: Messaging & Event-Driven Integration
* Event design (facts vs commands), ordering/idempotent consumers
* DLQ, retries, poison message handling
* Exactly-once myths and practical at-least-once designs
* Exercise: design a robust order->payment->shipping event flow

## 101-06: Testing as Architecture Feedback (No Tool Deep Dive)
* Why test boundaries are architecture boundaries
* Contract boundaries as anti-coupling mechanism
* Failure-path verifiability: timeouts, duplicates, out-of-order events
* Observability as testability for distributed systems
* Exercise: identify architecture risks and define evidence needed to trust the system

## 101-07: Deployment Safety & Progressive Delivery
* Blue/green, canary, feature flags, rollback strategy
* Migration playbooks (expand/contract, backfill)
* Release guardrails with SLOs and auto-abort
* Exercise: run a simulated canary incident response

## 101-08: Security & Multi-Tenancy Fundamentals
* Service-to-service authn/authz (JWT/mTLS basics)
* Secrets management, least privilege, auditability
* Tenant isolation patterns and noisy-neighbor controls
* Exercise: threat-model a microservice boundary

## 101-09: Performance & Capacity for Real Traffic
* Latency budgeting across call chains
* Concurrency limits, queue sizing, cache strategy
* Load test basics and bottleneck diagnosis
* Exercise: tune an overloaded dependency graph

---

If you want to keep it compact, a strong 6-session path is:

101-01 correctness
101-02 resilience/observability
101-03 data patterns
101-04 API evolution
101-05 messaging
101-06 testing as architecture feedback
If you want, I can draft outline-03.md next in the same style as outline-01/02 so the series continues seamlessly.

