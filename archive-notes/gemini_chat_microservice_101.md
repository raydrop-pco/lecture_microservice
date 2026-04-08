# Conversation: Microservice 101 Series

**User:**
I published microservice 101 series for entry-level engineers in my team. Session 101-01, and 101-02 are attached. I would like to keep dropping continuous contents. Propose the idea of further contents.
*(Note: Attachments 101-01 and 101-02 were referenced but the text content focuses on the subsequent proposal)*

**Gemini:**
The first two sessions lay a rock-solid foundation. Since 101-01 (Correctness/Idempotency) and 101-02 (Stability) deal with the "survival" of a single request, the next logical step is to zoom out to how multiple services interact and own their "truth."

Here is a proposal for the next four sessions, designed to be practical and punchy for entry-level engineers:

#### **101-03: Communication Patterns — Talk is Cheap, Coupling is Expensive**
*   **The Core Story:** Why you can't just call another service's function.
*   **Key Concepts:**
    *   **Synchronous (REST/gRPC):** High coupling, immediate feedback, but risks cascading slowness.
    *   **Asynchronous (Messaging/Queues):** Decoupling, high throughput, but introduces the complexity of eventual consistency.
*   **Hands-on Exercise:** Map out a "User Registration" flow using synchronous APIs vs. a message queue.

#### **101-04: Data Boundaries — Why You Can’t Just "Join" Anymore**
*   **The Core Story:** The "Shared Database" trap—how one small schema change in Service A breaks Service B, C, and D.
*   **Key Concepts:**
    *   **Data Encapsulation:** Why services should only "talk" to their own database.
    *   **API Composition vs. Join:** How to get data from two places at once.
    *   **Tenant Isolation:** Briefly introducing why keeping data separate matters for security.
*   **Hands-on Exercise:** Design a "Project Dashboard" that joins data from HR and Finance services using API calls.

#### **101-05: The "Buy vs. Build" Mindset — Using the Cloud as Your Lever**
*   **The Core Story:** Building a custom file-processing engine vs. using managed workflow tools and cloud storage.
*   **Key Concepts:**
    *   **Managed Services:** When to use AWS Step Functions or Azure Logic Apps instead of custom glue code.
    *   **Valet Key Pattern:** Securely offloading heavy lifting (like file uploads) directly to cloud storage.
*   **Hands-on Exercise:** Compare the "Build" route (custom servers) vs. the "Buy" route (Pre-signed URLs and serverless functions).

#### **101-06: API Evolution — How to Change Without Breaking the World**
*   **The Core Story:** You updated a field name from `user_id` to `account_id`, and suddenly the mobile app crashes for 50,000 people.
*   **Key Concepts:**
    *   **Expansion & Contraction Pattern:** How to safely migrate fields.
    *   **Versioning Strategies:** Headers vs. URL paths.
    *   **Backward Compatibility:** Why you should never "break the contract."
*   **Hands-on Exercise:** Design a deployment plan that allows old and new clients to coexist during a field refactor.

**Suggested Roadmap:**
| Session | Focus | Primary Goal |
| :--- | :--- | :--- |
| **101-01** | Correctness | Handling retries & async data safely. |
| **101-02** | Stability | Keeping the lights on when things fail. |
| **101-03** | Interaction | Choosing the right communication style. |
| **101-04** | Data | Living without SQL joins across services. |
| **101-05** | Leverage | Choosing managed services over custom infra. |
| **101-06** | Evolution | Managing change in a distributed environment. |

---

**User:**
the following is the idea from other AI. are there any takeaway from them? I love your 101-03 contents, but not sure it is the best position in our series.
---
The most natural next lessons are:
**101-03: Data Modeling & Storage Patterns for Microservices**
* DB-per-service, schema ownership, reference by ID
* Transaction boundaries, outbox/inbox, CDC basics
* Query patterns (API composition, CQRS-lite)
**101-04: API Design for Evolution**
* Backward compatibility, versioning strategies
* Error model, pagination/filtering consistency
* Contract testing and consumer-driven contracts

---

**Gemini:**
That other AI list is quite comprehensive, though it leans heavily toward "Day 2" operations. The biggest takeaway is **contextualizing the "Why" before the "How."** Moving **Data Modeling** to the 101-03 slot actually resolves your hesitation about the "Communication Patterns" session.

**Refined "Master Roadmap" (101-03 to 101-06):**

#### **101-03: Data Ownership & Boundaries**
*   **Focus:** How to live without shared databases.
*   **Key Lessons:** The "Database-per-service" rule, referencing by ID instead of Foreign Keys, and the basics of multi-tenant isolation.
*   **Exercise:** Split a single "Customer+Order" table into two services and determine how the Order service gets the Customer's name without a JOIN.

#### **101-04: API Evolution & Contracts**
*   **Focus:** Changing your service without breaking the team next door.
*   **Key Lessons:** Backward compatibility, the "Expand & Contract" pattern, and using versioning.
*   **Exercise:** Refactor an API field name (e.g., `user_id` to `account_id`) in a simulated live environment.

#### **101-05: Messaging & The "Buy vs. Build" Lever**
*   **Focus:** Moving from "Waiting for a response" to "Queuing a task."
*   **Key Lessons:** Using managed services (like SQS or Service Bus) instead of custom polling logic.
*   **Exercise:** Design a flow where a "File Upload" triggers a "Virus Scan" asynchronously using a queue.

#### **101-06: Production Safety & Multi-Tenancy**
*   **Focus:** Shipping safely and keeping users separated.
*   **Key Lessons:** Feature flags, canary releases, and Row Level Security (RLS) for multi-tenant PostgreSQL.
*   **Exercise:** Enable a "New Feature" for only 10% of users and roll it back instantly when errors spike.

**Comparison: Why this order works**
By moving **Data (101-03)** before **Messaging (101-05)**, you give them a place to *store* state before you teach them how to *move* state between services. 

**Would you like me to draft the "Exercise: Split a Monolith Table" for the new 101-03 session?**
