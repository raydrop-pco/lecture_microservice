# Microservices Data Ownership and Read/Write Patterns

## Slide 1
**<Slide Title>**  
Why Data Gets Harder in Microservices

**<Key message>**  
Once services own their own data, both reading and writing across the system become design problems.

**<Slide content>**  
- In a monolith, one database often makes reads and writes look straightforward.
- In microservices, each service should own its own source of truth.
- This improves autonomy and reduces direct database coupling.
- But it creates two new questions:
  - How do we assemble data for reads?
  - How do we propagate changes to derived or replicated data?

**<Illustration (if helpful)>**  
A simple diagram:
- Monolith: UI -> App -> One DB
- Microservices: UI -> Catalog / Price / Inventory, each with its own DB

**<Speaker notes>**  
Start from a familiar monolith mental model. Emphasize that microservices do not remove data problems; they move them into explicit design decisions. The key setup for the rest of the lecture is: once ownership is split, both read paths and write propagation need patterns.


## Slide 2
**<Slide Title>**  
First Principle: Split Data Ownership

**<Key message>**  
The main rule is not “split tables mechanically,” but “define a clear owner for each business fact.”

**<Slide content>**  
- Example ownership split:
  - Catalog owns product description and attributes
  - Price owns price and promotion rules
  - Inventory owns stock availability
- Each service owns its own write model and persistence.
- Other services should not update that data directly in the owner’s database.
- This avoids hidden coupling at the database level.

**<Illustration (if helpful)>**  
Three boxes:
- Catalog -> product data
- Price -> pricing data
- Inventory -> stock data

**<Speaker notes>**  
Clarify that this is about ownership, not about forcing one giant table to be physically decomposed without thought. The point is to establish clear responsibility and boundaries around change.


## Slide 3
**<Slide Title>**  
What New Questions Appear?

**<Key message>**  
Once ownership is split, read-side and write-side questions naturally diverge.

**<Slide content>**  
- Read-side question:
  - “How do I show one screen that needs data from multiple services?”
- Write-side question:
  - “If I keep a denormalized or replicated view, how does it get updated?”
- These are normal consequences of proper service boundaries.
- We need patterns, not shortcuts that break ownership.

**<Illustration (if helpful)>**  
A forked diagram:
- Data ownership split
  - Left: Read challenge
  - Right: Write propagation challenge

**<Speaker notes>**  
This slide is the transition. Keep it simple: splitting ownership is correct, but it introduces new operational and architectural questions. That is expected, not a sign that the service split was wrong.


## Slide 4
**<Slide Title>**  
Read Patterns: Gather Data on Demand

**<Key message>**  
One class of read pattern assembles the response at request time.

**<Slide content>**  
- Common options:
  - Direct synchronous service-to-service calls
  - API Composition in a backend layer
  - Client-side composition in the frontend
- Strengths:
  - Fresh data
  - No extra copy to maintain
- Trade-offs:
  - Higher latency
  - More runtime dependencies
  - More complex failure handling

**<Illustration (if helpful)>**  
Product Detail Page example:
UI -> Aggregator -> Catalog + Price + Inventory

**<Speaker notes>**  
Do not over-explain each pattern. The goal is to show that these all belong to the “assemble now” family. The main trade-off is freshness versus latency and dependency complexity.


## Slide 5
**<Slide Title>**  
Read Patterns: Prepare Data Ahead of Time

**<Key message>**  
Another class of read pattern prepares a read-friendly copy before the request arrives.

**<Slide content>**  
- Common options:
  - Cache
  - Aggregated read view
  - Search index or specialized read store
- Strengths:
  - Lower latency
  - Simpler read path
  - Often better for high-traffic screens
- Trade-offs:
  - Extra storage
  - Refresh logic required
  - Data may be temporarily stale

**<Illustration (if helpful)>**  
Product Detail View store fed by Catalog, Price, and Inventory updates

**<Speaker notes>**  
Frame this as “pre-compute or pre-position data for reads.” This is where many teams start introducing derived data. Do not introduce CQRS terminology here unless needed later.


## Slide 6
**<Slide Title>**  
Write-Side Question: How Do Derived Views Stay Updated?

**<Key message>**  
Once we create a read-friendly copy, we need an explicit update strategy.

**<Slide content>**  
- Typical update approaches:
  - Immediate local update
  - Scheduled refresh
  - Batch refresh
  - Event-driven asynchronous update
- The question is not only “how to notify,” but “how and when to reflect state changes.”
- The right choice depends on freshness requirements and operational complexity.

**<Illustration (if helpful)>**  
Order updated -> update strategies -> read view refreshed

**<Speaker notes>**  
* "Use “Immediate local update” to mean updating locally owned derived data within the same service flow. It does not mean distributed transactions across services."
* "The core point is that denormalized or replicated data does not maintain itself; the team must choose an explicit reflection strategy."
* "Use “reflect state changes” rather than only “send notifications.” The important idea is that denormalized or replicated data does not maintain itself; someone must own and operate the update path."


## Slide 7
**<Slide Title>**  
How to Choose Among the Patterns

**<Key message>**  
Choose patterns by business requirements, not by architectural fashion.

**<Slide content>**  
Evaluate each option using:
- Data freshness requirements
- End-to-end latency targets
- Runtime dependency tolerance
- Implementation and operational complexity
- Traffic shape and scaling needs
- Failure and recovery behavior

**<Illustration (if helpful)>**  
A simple decision table with columns:
Freshness / Latency / Complexity / Dependency / Typical fit

**<Speaker notes>**  
This is one of the most important slides. Teams often jump to a pattern because it sounds modern. Keep the message practical: choose based on business and operational constraints.


## Slide 8
**<Slide Title>**  
Be Precise About Ownership of Copies

**<Key message>**  
Any cache, replica, or aggregated view must have a clearly defined owner and purpose.

**<Slide content>**  
- Always define:
  - Who owns the original data?
  - Who owns the derived copy?
  - Which use case the copy exists for
  - Who is responsible for refresh and correctness
- Good question:
  - “Whose read problem does this copy solve?”
- Without this clarity, copied data becomes unmanaged coupling.

**<Illustration (if helpful)>**  
Original owner services feeding a “Product Detail View” owned by a read/experience layer

**<Speaker notes>**  
This slide is a guardrail. Many problems come from creating convenient copies with no clear owner. Stress that a derived view is not free; it is a product with an owner, purpose, and lifecycle.


## Slide 9
**<Slide Title>**  
Expect Eventual Consistency in Some Designs

**<Key message>**  
If derived data is updated later, temporary inconsistency is a design consequence, not an accident.

**<Slide content>**  
- If updates are asynchronous, the copy may lag behind the source of truth.
- This means some read paths are eventually consistent.
- Design questions:
  - How stale is acceptable?
  - Which screens need near-real-time data?
  - How do users experience lag or partial updates?
- Treat consistency level as an explicit design decision.

**<Illustration (if helpful)>**  
Timeline:
Write committed -> event emitted -> read view updated a bit later

**<Speaker notes>**  
Keep this grounded. Eventual consistency does not mean “anything goes.” It means the team must explicitly decide where temporary divergence is acceptable and how that affects user experience.


## Slide 10
**<Slide Title>**  
Summary

**<Key message>**  
Microservices push teams to separate data ownership first, then intentionally choose read and write patterns around that boundary.

**<Slide content>**  
- Start with clear ownership of business facts.
- Expect separate read-side and write-side questions to appear.
- Read patterns fall into two families:
  - Gather on demand
  - Prepare ahead of time
- Derived copies need:
  - Clear ownership
  - Explicit refresh strategy
  - Conscious consistency decisions
- Avoid database-level shortcuts that create hidden coupling across services.

**<Illustration (if helpful)>**  
One summary diagram:
Owned source data -> read patterns -> derived views -> consistency decisions

**<Speaker notes>**  
Close by reinforcing the mental model: ownership first, then read/write patterns. Mention that database-level materialized views or cross-service joins may look convenient, but they often undermine the service boundaries the architecture is trying to create.

