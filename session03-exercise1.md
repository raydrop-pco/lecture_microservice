# Session 3 — Exercise 1: Master Data Decomposition (25–30 min)

## Setup (what attendees get)

You're tasked with breaking down a large shared database table into independently owned, service-specific schemas.

**Scenario**

* Your company inherited a monolith with a `products` table (~60 columns).
* Five services currently read/write to this table: **Catalog**, **Pricing**, **Inventory**, **Shipping**, and **Finance**.
* There is no dedicated "product data" team — each service added columns as they needed them.
* Teams want to move to microservices with **independent deployments and clear ownership boundaries**.
* Requirement: **Design the ownership split and identify the hard cases** where a single attribute could reasonably belong to multiple domains.

**Assumptions**

* You can model each service's data independently.
* You can call other services' APIs to resolve references.
* Data consistency across services will be eventual (via events, not global transactions).
* Caching and denormalization are allowed to manage latency.

---

# Attendee Worksheet (printable)

### Part A — Diagnose Ownership (10 min)

Below are the major column groups from the shared `products` table (grouped by logical domain). For each, answer:

1. **Which business function owns / creates / maintains this data?**
2. **Which service should be the system of record?**
3. **Which other services only *read* it — and could resolve it on demand instead of denormalizing?**

---

#### Column Group 1: Presentation

**Columns:** `name`, `description`, `images` (JSON), `slug`, `tags`, `seo_title`, `seo_description`

* Who owns this? ___________________________________
* System of record service? ________________________
* Services that read it? ____________________________
* On-demand vs. denormalized? ______________________

**Notes:**
* How often does it change?
* Does it need to be consistent in real-time across all services?

---

#### Column Group 2: Pricing

**Columns:** `list_price`, `sale_price`, `discount_rules` (JSON), `currency`, `tax_applicable`, `bulk_pricing` (JSON)

* Who owns this? ___________________________________
* System of record service? ________________________
* Services that read it? ____________________________
* On-demand vs. denormalized? ______________________

**Notes:**
* Does the Inventory service need the current price to allocate stock?
* Does a product catalog display need real-time pricing?

---

#### Column Group 3: Stock & Inventory

**Columns:** `sku`, `quantity_on_hand`, `reorder_threshold`, `warehouse_id`, `inventory_status`, `last_stock_checked_at`

* Who owns this? ___________________________________
* System of record service? ________________________
* Services that read it? ____________________________
* On-demand vs. denormalized? ______________________

**Notes:**
* Does Catalog publish product availability to the web?
* How often does stock level change?

---

#### Column Group 4: Logistics & Shipping

**Columns:** `weight_kg`, `dimensions_cm` (JSON), `volume_m3`, `hazmat_flag`, `carrier_code`, `shipping_class`

* Who owns this? ___________________________________
* System of record service? ________________________
* Services that read it? ____________________________
* On-demand vs. denormalized? ______________________

**Notes:**
* When do these attributes change?
* Which services reference them, and how often?

---

#### Column Group 5: Financial

**Columns:** `cost_price`, `cost_currency`, `tax_class_id`, `country_of_origin`, `accounting_code`, `margin_threshold`

* Who owns this? ___________________________________
* System of record service? ________________________
* Services that read it? ____________________________
* On-demand vs. denormalized? ______________________

**Notes:**
* Does Pricing service need cost_price to calculate discounts?
* Is this sensitive data that requires financial authorization?

---

#### Column Group 6: Metadata & Status

**Columns:** `status` (active/inactive/discontinued), `created_at`, `updated_at`, `version`, `updated_by`, `soft_delete`

* Who owns `status`? _______________________________
* Who owns lifecycle/audit fields? ___________________
* What if Pricing and Inventory disagree on `status`? __________________

---

### Part B — Draw the Split (10 min)

Now design the ownership model. For each service, fill in:

#### Service 1: **Catalog**

* **Owns these columns:** ________________________
* **Schema name / table name:** ___________________
* **Primary key:** ________________________________
* **Unique constraints:** __________________________
* **API response** (what the Catalog service returns about a product):
  ```json
  {
    "product_id": "...",
    "name": "...",
    ...
  }
  ```

---

#### Service 2: **Pricing**

* **Owns these columns:** ________________________
* **Schema name / table name:** ___________________
* **Primary key:** ________________________________
* **Foreign references** (which service owns the ID it references?):
  * product_id → Catalog
  * ___________________________________________
* **API response** (what the Pricing service returns):
  ```json
  {
    "product_id": "...",
    "list_price": "...",
    ...
  }
  ```

---

#### Service 3: **Inventory**

* **Owns these columns:** ________________________
* **Schema name / table name:** ___________________
* **Primary key:** ________________________________
* **Foreign references:**
  * product_id → Catalog
  * ___________________________________________
* **API response:**
  ```json
  {
    "product_id": "...",
    "sku": "...",
    ...
  }
  ```

---

#### Service 4: **Shipping**

* **Owns these columns:** ________________________
* **Schema name / table name:** ___________________
* **Primary key:** ________________________________
* **Foreign references:**
  * product_id → Catalog
  * ___________________________________________
* **API response:**
  ```json
  {
    "product_id": "...",
    "weight_kg": "...",
    ...
  }
  ```

---

#### Service 5: **Finance**

* **Owns these columns:** ________________________
* **Schema name / table name:** ___________________
* **Primary key:** ________________________________
* **Foreign references:**
  * product_id → Catalog
  * ___________________________________________
* **API response:**
  ```json
  {
    "product_id": "...",
    "cost_price": "...",
    ...
  }
  ```

---

### Part C — Identify Ambiguities (8 min)

For each "hard case," decide:

1. **`product_status` (active / inactive / discontinued)**
   * Does every service need the same view of status?
   * What if Pricing wants to mark a product "inactive for discounting" while Catalog still shows it?
   * Who is the source of truth? ___________________
   * How do consumers learn about status changes? ___________

2. **`SKU`**
   * Is it a Catalog concept (a naming/identification scheme) or an Inventory concept (an operational identifier)?
   * What if different services generate SKUs differently?
   * Owner: ( ) Catalog  ( ) Inventory  ( ) Both (different tables)

3. **`created_at` and `updated_at`**
   * Does each service track creation/update of *their own data*, or do they share a global timeline?
   * Example: if Pricing updates price, should Catalog's `updated_at` also change?
   * Answer: ___________________________________

4. **A Product Detail Page (Web/API)**
   * Request: `GET /product/{product_id}`
   * Must return: name, image, price, stock status, weight, tax_class
   * How do you fetch all of this without calling 5 different services every time?
   * Strategy (choose one):
     * ( ) API composition: call all 5 in parallel, merge in real-time
     * ( ) Denormalization: Catalog maintains a denormalized copy of [price, stock, weight]
     * ( ) CQRS-lite: maintain a separate read-optimized projection table
   * Why? ______________________________

---

### Part D — Bonus (if time)

5. **Migration path:**
   * You have a live system currently using the shared `products` table.
   * Walk through a 3-step migration plan to split it without downtime:
     * Step 1: ______________________________
     * Step 2: ______________________________
     * Step 3: ______________________________

---

---

# Session 3 — Exercise 1 Facilitation Guide

## Objective

**Design a data ownership model that breaks the shared-database coupling while exposing the complexity that ownership creates.**

Key learning: Ownership boundaries are not just technical cuts; they reflect business domains. If you can't name a team responsible for a piece of data, it doesn't belong in a single table.

---

## Timing (25–30 min)

* **2 min:** Intro + scenario setup
* **10 min:** Part A (diagnose ownership per column group)
* **10 min:** Part B (design service schemas and APIs)
* **8 min:** Part C (hard cases + decision-making)
* **5 min:** Share-out + debrief

---

## Facilitation Strategy

### Part A — Diagnose Ownership (10 min)

**Opening framing:**

> "For each column group, ask: 'Which business team creates this data? Which service is responsible for keeping it accurate?' If you can't name a team, it probably doesn't belong in one place."

**Key questions to ask groups:**

* **On Presentation (name, description, images):**
  * "Who in the company decides what the product is called? Is it the same team that decides pricing?"
  * "How often does this change? Daily? Once a month?"

* **On Pricing:**
  * "Does Inventory need the current price to make inventory decisions? Or does Pricing own price, and Inventory just stores it?"
  * "What if Pricing wants a seasonal sale, but Inventory should still stock at full quantity?"

* **On Inventory:**
  * "Who reorders stock? Is that the same team as the one that decides whether to label a product as 'low stock'?"
  * "If you have a multi-warehouse setup, does one warehouse own all stock or share it?"

* **On Shipping:**
  * "Does weight change? If not, why does Finance need it?"
  * (Answer: Finance calculates dimensional weight / shipping cost impact)

* **On Finance (cost_price, tax_class):**
  * "This is usually sensitive. Who has access to cost_price? Should Catalog or Pricing even have it?"
  * "Does Pricing need cost to calculate margins? Or is that a Finance responsibility?"

* **On Status (tricky):**
  * "If Catalog marks a product discontinued, should Pricing still allow orders? Should Inventory still show it?"
  * "This is a real pain point. Different teams often have different 'active' criteria."

---

### Part B — Draw the Split (10 min)

**Opening framing:**

> "Now translate ownership into schemas. For each service: what table do they own, what's their API surface, and what IDs do they store to reference other services?"

**Key questions:**

* **On foreign references:**
  * "You're in the Pricing service. You need to know which Product this pricing applies to. Do you store the product_id? Yes."
  * "Do you store a copy of the product name in the pricing table? Only if you're denormalizing for a specific query reason."

* **On the "all services reference product_id to Catalog":**
  * "Why does Catalog own product_id and not Pricing? Because product_id is the identity of the thing itself—the business concept."
  * "If Pricing 'owned' product_id, then Inventory couldn't even name a product without Pricing creating it first. That's coupling."

* **On API responses:**
  * "Your Pricing API returns price. Should it also return the product name from Catalog? Not in the base case—reference by ID. If you *need* the name on every call, that's a sign you should denormalize it in Pricing's own table."

---

### Part C — Identify Ambiguities (8 min)

**These are the real debates:**

#### Case 1: `product_status`

**The hard reality:**
* Catalog says a product is "active" (visible on the website).
* Finance says it's "discontinued" (no new orders, but fulfill existing ones).
* Pricing wants to mark it "inactive for discounts" (keep inventory, disable promotional pricing).

**Facilitation:**
* Ask: "Who makes the final call? If Catalog is the UI, does Catalog have veto? Or does status mean different things to different teams?"
* **Possible answer:** No single `product_status`. Each service has its own lifecycle tracking. Inventory can refuse to stock discontinued items; Pricing can apply rules; Catalog can set visibility independently.
* Or: Catalog is the source of truth for "global" status, and others cache/subscribe to it.

#### Case 2: `SKU`

**The question:**
* Is SKU a Catalog concept (a user-friendly identifier like "WIDGET-2024-RED") or an Inventory concept (an internal warehouse code)?

**Facilitation:**
* In many real systems, **Inventory owns SKU** because it's the warehouse/supply-chain identifier.
* Catalog references SKU as a denormalized field for product listings.
* But in others, Catalog owns both the marketing name and a catalog-friendly identifier.
* **Key insight:** There's no universally right answer. It depends on your business. The exercise is to make the choice explicit.

#### Case 3: `created_at / updated_at`

**The debate:**
* Does "updated_at" reflect when the Product row changed, or when *this service's view* of it changed?

**Facilitation:**
* If shared, you're back to a shared `updated_at` column, which couples all services.
* **Better:** Each service tracks when *they* last modified their data. Catalog has `catalog_updated_at`, Pricing has `pricing_updated_at`.
* When Pricing changes, Catalog's `catalog_updated_at` does NOT change. Catalog learns about Pricing changes via events, not via a shared `updated_at`.

#### Case 4: Product Detail Page

**The constraint:**
* Must render: name + image + price + stock + weight + tax in one screen.
* That's 4+ services. Do you call all 4 on every page load?

**Facilitation:**
* **API composition** (naive): Yes, call all 4 in parallel. Pros: always fresh, no cache invalidation. Cons: 4 network hops, latency.
* **Denormalization in Catalog**: Catalog maintains a *denormalized* copy of price + stock + weight + tax, updated via events from each service. Pros: single call, consistent latency. Cons: eventual consistency (pricing may lag), more complex to keep in sync.
* **CQRS-lite read model**: Separate database optimized for this query, kept fresh by events. Pros: independent scaling, clean separation. Cons: operational overhead.
* **Takeaway:** No free lunch. Each approach trades freshness for latency or operational complexity. Part of mature microservices is choosing consciously.

---

### Part D — Bonus: Migration Path (if time)

**This is hard in practice. Some ideas:**

1. **Expand phase:** Add new schema/table in the destination service. Both new and old writes go to both places. Dual-write temporarily.
2. **Migrate data:** Copy existing data from shared table to destination service tables.
3. **Cut over:** Stop writing to shared table. Read from new tables.
4. **Contract phase:** Drop old table once you're confident.

**Or simpler:** Shadow write to new tables in parallel, then flip reading once data is consistent.

---

## Common Pitfalls & How to Address Them

| Pitfall | What attendees often do | How to redirect |
|---------|--------------------------|-----------------|
| **"We have to split it exactly as the ER diagram looks"** | Treat table structure as law, not business structure | Ask: "Which team maintains each column? Not the table—the column." |
| **"Cost price belongs in Finance, so Finance gets the whole financial table"** | Over-group related columns | "Does Accounting own SKU mapping? Does Accounting update images?" |
| **"Let's just share a read-only replica of the products table"** | Recreate shared DB as a "safe" CQRS | "You're back to a shared schema. Something changed in Catalog. Did Finance cache the right version?" |
| **"If Pricing needs a product name, it should join the Catalog table"** | Reintroduce cross-service JOINs at the data layer | "That's a cross-service foreign key. Who decides if the name changed—Catalog or Pricing? One must own it." |
| **"Status belongs to nobody—it's metadata"** | Leave ownership ambiguous | "If everyone reads it, nobody updates it. Then it gets stale. Assign an owner." |
| **"The product detail page takes too long with 5 service calls"** | Accept the latency without exploring caching | "True. But before adding a cache, design the invalidation strategy. How do you know when to refresh?" |

---

## Debrief Questions (5 min)

Ask the room / breakout groups:

1. **"Did you find columns that could belong to multiple services?"**
   * This is the bounded-context insight: one data attribute can have different meanings in different domains.

2. **"Which query pattern did you choose for the product detail page, and why?"**
   * Look for groups that chose different answers. Discuss the tradeoffs.

3. **"What surprised you about the hard cases (status, SKU, created_at)?"**
   * Often: "We realized we've been assuming Catalog owns everything."

4. **"How does this compare to how your current system works?"**
   * If they're living the shared DB, this should sting. That's good.

5. **"If you had to migrate your real system this way, what's your biggest blocker?"**
   * Honest answer often: "The schema is already 10 years old, and changing it is politically hard."
   * Frame response: "You're right. That's why the exercise is about clarity first—then migration strategy follows."

---

# Model Answers (don't reveal all at once—guide toward them)

**Complete ownership mapping (all as-is `products` attributes):**

### 1) Catalog Service (source of truth for product identity, merchandising, and discovery)

**Owns:**
* `product_id`, `product_uuid`, `internal_code`
* `name`, `slug`, `description`, `short_description`, `long_description`
* `meta_title`, `meta_description`, `meta_keywords`
* `primary_image_url`, `thumbnail_image_url`, `gallery_images`, `video_url`
* `category_id`, `subcategory_id`, `tags`, `collection_ids`
* `is_published`, `publication_date`
* `status` (if your org chooses a single global lifecycle status)
* `brand_id`, `manufacturer_id`, `is_digital_product`, `warranty_months`, `return_eligible`

**Denormalizes (optional):**
* `list_price`, `sale_price` (from Pricing)
* `inventory_status` (from Inventory)
* `shipping_class` (from Shipping)

### 2) Pricing Service (source of truth for sell price and discount policy)

**Owns:**
* `list_price`, `sale_price`
* `sale_price_start_date`, `sale_price_end_date`
* `currency_code`, `discount_percentage`
* `discount_rules`, `bulk_pricing`
* `tax_class_id`, `tax_applicable`

**Reads on demand / subscribes to:**
* Product identity fields from Catalog (`product_id`, `name`, `slug` as needed)
* Cost data from Finance for margin guardrails (if required)

### 3) Inventory Service (source of truth for stock and replenishment)

**Owns:**
* `sku`
* `quantity_on_hand`, `reserved_quantity`
* `reorder_level`, `reorder_threshold`, `reorder_quantity`
* `warehouse_id`, `warehouse_location`
* `inventory_status`, `last_stock_checked_at`
* `supplier_id`, `supplier_sku`, `lead_time_days`

**Reads on demand / subscribes to:**
* Product identity from Catalog
* Optional pricing signals from Pricing (for inventory prioritization policies)

### 4) Shipping Service (source of truth for shipment operations + product transport attributes)

**Owns:**
* `weight_kg`, `weight_lb`
* `length_cm`, `width_cm`, `height_cm`, `dimensions_cm`
* `volume_m3`
* `hazmat_flag`, `hazmat_class`, `hazmat_description`
* `shipping_class`, `carrier_code`
* `is_fragile`, `requires_refrigeration`, `max_items_per_shipment`
* `country_of_origin`, `harmonized_tariff_code`

**Reads on demand / subscribes to:**
* `sku` (if routing logic depends on Inventory-owned SKU)
* Product display fields from Catalog for labels/documents

**Datastore split inside Shipping Service (important):**
* `shipment_ops_db` for shipment execution (`when/where/how`, status, tracking events, delivery attempts).
* `product_logistics_db` for product-level transport attributes (dimensions, hazmat, shipping class, carrier rules).
* Same service, two bounded data concerns; keep schemas separate to avoid coupling profile data with transaction flows.

### 5) Finance Service (source of truth for cost and accounting)

**Owns:**
* `cost_price`, `cost_currency`
* `gross_margin_percentage`, `margin_threshold`
* `accounting_code`, `cost_center_id`

**Reads on demand / subscribes to:**
* Sell price from Pricing for margin analysis
* Product identity from Catalog for reporting dimensions

### Implementation image: sample datastore/table names by domain

Use these as reference names in design discussions and whiteboard diagrams.

| Domain / Service | Sample datastore | Sample tables | Purpose |
|---|---|---|---|
| Catalog | `catalog_db` | `catalogs`, `catalog_media`, `catalog_categories`, `catalog_collections` | Product identity, descriptions, SEO, media, taxonomy |
| Pricing | `pricing_db` | `prices`, `price_rules`, `price_tiers`, `price_tax_policies` | Sell prices, discount rules, tax applicability |
| Inventory | `inventory_db` | `inventory_levels`, `inventory_items`, `inventory_reservations`, `reorder_policies`, `suppliers` | Stock counts, SKU operations, warehouse/replenishment |
| Shipping | `shipment_ops_db` + `product_logistics_db` | `shipments`, `shipment_events`, `shipment_tracking`, `delivery_attempts`; `product_shipping_profiles`, `product_hazmat_profiles`, `carrier_rule_mappings` | One service with two datastores: shipment transaction flows + product-level logistics attributes |
| Finance | `finance_db` | `product_costs`, `margin_policies`, `accounting_mappings`, `cost_centers` | Cost, margin, accounting attribution |
| Reviews (projection) | `reviews_read_db` | `product_reviews_summary` | `avg_rating`, `review_count` read model |
| Analytics (projection) | `analytics_read_db` | `product_popularity` | `popularity_score`, `trending_flag` read model |
| Search / Product Detail Read Model | `product_read_db` | `product_detail_projection`, `product_search_projection` | CQRS-lite projection for fast product pages/search |
| Cross-service Audit (optional) | `audit_db` | `domain_event_log`, `entity_change_audit` | Global timeline/audit without shared mutable product row |

**Naming notes:**
* Keep names plural and domain-specific (example: `prices`, `inventory_levels`, `product_shipping_profiles`).
* Avoid generic names like `products_master` or `common_products` that invite shared ownership.
* In this scenario, Shipping is a single service with two datastores; still distinguish product logistics attributes from shipment transactions to avoid mixing master data with operational flows.
* If your org has naming standards, apply them consistently (snake_case, prefixes, or schema-per-domain).

### 6) Cross-cutting lifecycle and audit fields (do not keep as one shared row)

**As-is fields:**
* `soft_delete`, `discontinued_date`
* `created_at`, `updated_at`, `created_by`, `updated_by`
* `version`, `last_modified_source`

**Model answer guidance:**
* Avoid one global copy of these in a shared table.
* Keep per-service metadata in each service's own table (example: `catalog_updated_at`, `pricing_updated_at`, `inventory_version`).
* If a global audit timeline is needed, build it as an event/audit stream, not a shared mutable row.

### 7) Fields that should move out of `products` into dedicated read/projection services

**As-is fields:**
* `avg_rating`, `review_count` -> Reviews read model
* `popularity_score`, `trending_flag` -> Analytics/Ranking read model

**Model answer guidance:**
* These are read-optimized projections, not product master ownership data.
* Publish events from source systems; update projection tables asynchronously.

### 8) Resolution for known ambiguities

**`status`:**
* Option A (simpler): Catalog owns global `status`; others consume as policy input.
* Option B (more explicit): each service has its own status (`catalog_status`, `inventory_status`, `pricing_status`) and no single shared `status`.

**`sku`:**
* Default model answer: Inventory owns `sku`.
* Acceptable alternative: Catalog owns merchandising SKU while Inventory owns operational SKU, with clear naming.

**`created_at` / `updated_at`:**
* Per-service timestamps only. Do not propagate one shared timestamp across all domains.

**Product detail page (`name + image + price + stock + weight + tax_class`):**
* Preferred model answer: CQRS-lite read projection fed by Catalog/Pricing/Inventory/Shipping events.
* Alternative for low scale: API composition with parallel calls.

### 9) Attribute-to-owner lookup table (quick facilitation reference)

| Attribute | Recommended owner | Notes |
|---|---|---|
| product_id | Catalog | Product identity root |
| product_uuid | Catalog | Public/global identifier |
| internal_code | Catalog | Internal merchandising code |
| name | Catalog | Product display naming |
| slug | Catalog | SEO/web URL identity |
| description | Catalog | Core merchandising text |
| short_description | Catalog | Listing/snippet text |
| long_description | Catalog | Detail page text |
| meta_title | Catalog | SEO metadata |
| meta_description | Catalog | SEO metadata |
| meta_keywords | Catalog | SEO metadata |
| primary_image_url | Catalog | Media for listing/detail |
| thumbnail_image_url | Catalog | Media for listing |
| gallery_images | Catalog | Rich media set |
| video_url | Catalog | Rich media |
| category_id | Catalog | Taxonomy ownership |
| subcategory_id | Catalog | Taxonomy ownership |
| tags | Catalog | Discovery/search tags |
| collection_ids | Catalog | Merchandising collections |
| list_price | Pricing | Base sell price |
| sale_price | Pricing | Effective discounted price |
| sale_price_start_date | Pricing | Promo validity |
| sale_price_end_date | Pricing | Promo validity |
| currency_code | Pricing | Commercial currency |
| discount_percentage | Pricing | Derived/managed pricing control |
| discount_rules | Pricing | Rule engine inputs |
| bulk_pricing | Pricing | Tier pricing policy |
| tax_class_id | Pricing | Checkout tax classification input |
| tax_applicable | Pricing | Pricing/tax applicability flag |
| country_of_origin | Shipping | Customs/compliance + shipping docs |
| harmonized_tariff_code | Shipping | Customs/compliance data |
| sku | Inventory | Operational stock identifier |
| quantity_on_hand | Inventory | Available stock |
| reserved_quantity | Inventory | Allocation management |
| reorder_level | Inventory | Replenishment policy |
| reorder_threshold | Inventory | Replenishment policy |
| reorder_quantity | Inventory | Replenishment policy |
| warehouse_id | Inventory | Stock location owner |
| warehouse_location | Inventory | Stock location details |
| inventory_status | Inventory | Stock state |
| last_stock_checked_at | Inventory | Stock verification timestamp |
| weight_kg | Shipping | Physical dimensions domain |
| weight_lb | Shipping | Physical dimensions domain |
| length_cm | Shipping | Physical dimensions domain |
| width_cm | Shipping | Physical dimensions domain |
| height_cm | Shipping | Physical dimensions domain |
| dimensions_cm | Shipping | Physical dimensions snapshot |
| volume_m3 | Shipping | Logistics calc input |
| hazmat_flag | Shipping | Transport safety classification |
| hazmat_class | Shipping | Transport safety classification |
| hazmat_description | Shipping | Transport safety details |
| shipping_class | Shipping | Carrier/routing class |
| carrier_code | Shipping | Carrier integration key |
| is_fragile | Shipping | Packing/handling policy |
| requires_refrigeration | Shipping | Cold-chain requirement |
| max_items_per_shipment | Shipping | Shipment constraint |
| cost_price | Finance | Cost of goods |
| cost_currency | Finance | Cost accounting currency |
| gross_margin_percentage | Finance | Margin control/analytics |
| margin_threshold | Finance | Margin guardrail policy |
| accounting_code | Finance | GL mapping |
| cost_center_id | Finance | Accounting ownership dimension |
| status | Catalog or per-service | Pick one global owner, or split statuses by domain |
| is_published | Catalog | Channel publication control |
| publication_date | Catalog | Publishing lifecycle |
| soft_delete | Catalog or per-service | Prefer per-service soft-delete flags |
| discontinued_date | Catalog or per-service | Usually tied to catalog lifecycle |
| created_at | Per-service metadata | Do not share one mutable timestamp |
| updated_at | Per-service metadata | Do not share one mutable timestamp |
| created_by | Per-service metadata | Actor within owning service context |
| updated_by | Per-service metadata | Actor within owning service context |
| version | Per-service metadata | Optimistic lock per bounded context |
| last_modified_source | Per-service metadata | Replace with event/audit stream when possible |
| supplier_id | Inventory | Supplier relationship for replenishment |
| supplier_sku | Inventory | Supplier catalog mapping |
| lead_time_days | Inventory | Procurement/replenishment input |
| brand_id | Catalog | Merchandising/brand taxonomy |
| manufacturer_id | Catalog or Inventory | Decide based on business ownership of manufacturer master |
| is_digital_product | Catalog | Product type classification |
| warranty_months | Catalog | Product offer policy |
| return_eligible | Catalog | Merchandising/returns policy |
| avg_rating | Reviews projection | Move to reviews read model |
| review_count | Reviews projection | Move to reviews read model |
| popularity_score | Analytics projection | Move to ranking/analytics read model |
| trending_flag | Analytics projection | Move to ranking/analytics read model |

---

## Handout Snippet (for participants to take home)

```markdown
# Master Data Diagnostics Checklist

When you find a large shared table, ask these questions:

1. **For each column group:**
   - Who creates it? (business role)
   - Who owns it? (service)
   - Who reads it? (all services or specific ones?)

2. **If multiple services read it:**
   - Can they call the owner on demand?
   - Or must they denormalize it (cache a copy)?

3. **If multiple services write it:**
   - Is it really multiple owners, or is one the source of truth?
   - If multiple, can they live in separate tables?

4. **If you can't answer "who owns this column?":**
   - It doesn't belong in a shared table.
   - Either: assign ownership, or: it shouldn't exist.

5. **Denormalization rule of thumb:**
   - If data changes every second: query it on demand.
   - If data changes once a month: cache it (subscribe to changes).
```

---

# Appendix: As-Is Products Table Schema

Below is the full schema of the shared `products` table you're decomposing. This is what students will see—a realistic monolith schema with ~60 columns accumulated over 10+ years.

## Table: `products`

```sql
CREATE TABLE products (
  -- Primary & Identity
  product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  product_uuid UUID UNIQUE NOT NULL,
  internal_code VARCHAR(50) UNIQUE,
  
  -- Presentation / Catalog (domains: name, description, images, searchability)
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(255) UNIQUE NOT NULL,
  description TEXT,
  short_description VARCHAR(500),
  long_description TEXT,
  meta_title VARCHAR(255),
  meta_description VARCHAR(500),
  meta_keywords VARCHAR(500),
  
  -- Images & Media
  primary_image_url VARCHAR(1000),
  thumbnail_image_url VARCHAR(1000),
  gallery_images JSON,  -- Array of image URLs
  video_url VARCHAR(1000),
  
  -- Categorization & Tags
  category_id BIGINT,
  subcategory_id BIGINT,
  tags JSON,  -- Array of tag strings
  collection_ids JSON,  -- Array of collection IDs
  
  -- Pricing (domain: price, discounts, rules)
  list_price DECIMAL(12,2) NOT NULL,
  sale_price DECIMAL(12,2),
  sale_price_start_date DATETIME,
  sale_price_end_date DATETIME,
  currency_code CHAR(3) DEFAULT 'USD',
  discount_percentage DECIMAL(5,2),
  discount_rules JSON,  -- Complex pricing rules (seasonal, volume-based, etc.)
  bulk_pricing JSON,  -- Pricing tiers for bulk orders
  
  -- Taxes & Regulatory
  tax_class_id BIGINT,
  tax_applicable BOOLEAN DEFAULT TRUE,
  country_of_origin CHAR(2),
  harmonized_tariff_code VARCHAR(20),
  
  -- Inventory / Stock (domain: SKU, quantity, warehouse, thresholds)
  sku VARCHAR(50) UNIQUE NOT NULL,
  quantity_on_hand INT NOT NULL DEFAULT 0,
  reserved_quantity INT DEFAULT 0,
  reorder_level INT,
  reorder_threshold INT,
  reorder_quantity INT,
  warehouse_id BIGINT,
  warehouse_location VARCHAR(100),
  inventory_status VARCHAR(50),  -- in_stock, low_stock, out_of_stock, discontinued
  last_stock_checked_at DATETIME,
  
  -- Shipping & Logistics (domain: weight, dimensions, hazmat, carrier)
  weight_kg DECIMAL(10,3),
  weight_lb DECIMAL(10,3),
  length_cm DECIMAL(10,2),
  width_cm DECIMAL(10,2),
  height_cm DECIMAL(10,2),
  dimensions_cm JSON,  -- Structured storage: {length, width, height}
  volume_m3 DECIMAL(10,5),
  hazmat_flag BOOLEAN DEFAULT FALSE,
  hazmat_class VARCHAR(20),
  hazmat_description TEXT,
  shipping_class VARCHAR(50),
  carrier_code VARCHAR(50),
  is_fragile BOOLEAN DEFAULT FALSE,
  requires_refrigeration BOOLEAN DEFAULT FALSE,
  max_items_per_shipment INT,
  
  -- Financial / Accounting (domain: cost, margin, accounting codes)
  cost_price DECIMAL(12,2),
  cost_currency CHAR(3) DEFAULT 'USD',
  gross_margin_percentage DECIMAL(5,2),
  margin_threshold DECIMAL(5,2),
  accounting_code VARCHAR(50),
  cost_center_id BIGINT,
  
  -- Product Status & Lifecycle (potential ambiguity!)
  status VARCHAR(50) NOT NULL DEFAULT 'active',  -- active, inactive, discontinued, archived
  is_published BOOLEAN DEFAULT FALSE,
  publication_date DATETIME,
  soft_delete BOOLEAN DEFAULT FALSE,
  discontinued_date DATETIME,
  
  -- Audit & Metadata
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  created_by VARCHAR(100),
  updated_by VARCHAR(100),
  version INT DEFAULT 1,
  last_modified_source VARCHAR(50),  -- 'catalog_ui', 'pricing_api', 'inventory_sync', etc.
  
  -- Additional Attributes (accumulated over time)
  supplier_id BIGINT,
  supplier_sku VARCHAR(50),
  lead_time_days INT,
  brand_id BIGINT,
  manufacturer_id BIGINT,
  is_digital_product BOOLEAN DEFAULT FALSE,
  warranty_months INT,
  return_eligible BOOLEAN DEFAULT TRUE,
  
  -- Denormalized / Cached Fields (questions for students)
  avg_rating DECIMAL(3,2),
  review_count INT,
  popularity_score INT,
  trending_flag BOOLEAN DEFAULT FALSE,
  
  -- Indexes for common queries
  KEY idx_slug (slug),
  KEY idx_sku (sku),
  KEY idx_status (status),
  KEY idx_category (category_id),
  KEY idx_created_at (created_at),
  KEY idx_updated_at (updated_at),
  KEY idx_warehouse (warehouse_id),
  KEY idx_supplier (supplier_id)
);
```

---

## Notes for Students

**Observations about this schema:**

1. **Multiple "owners" writing to the same table:**
   * Catalog Service: updates name, description, images, category, tags
   * Pricing Service: updates list_price, sale_price, discount_rules, tax_class_id
   * Inventory Service: updates sku, quantity_on_hand, reorder_level, warehouse_id, inventory_status
   * Shipping Service: updates weight, dimensions, hazmat_flag, carrier_code
   * Finance Service: updates cost_price, margin_threshold, accounting_code
   * Everyone: access created_at, updated_at, status

2. **The `status` column problem:**
   * Used by all five services, but does it mean the same thing to all of them?
   * Has Pricing wants to mark a product "inactive for new orders" but Inventory is still re-stocking it.

3. **Denormalized fields:**
   * `avg_rating`, `review_count`, `popularity_score` — these should live in a Reviews/Analytics service.
   * Why are they here? Probably added for query performance without addressing the root coupling.

4. **Audit trails:**
   * `created_by`, `updated_by`, `last_modified_source` show multiple services are writing.
   * A single `updated_at` timestamp can't represent when each domain's data last changed.

5. **Foreign keys to other services:**
   * `supplier_id`, `brand_id`, `manufacturer_id` — these are IDs from other services's tables.
   * No explicit foreign key constraint (common in monoliths). Which service owns these relationships?

6. **The "version" column:**
   * Hints at optimistic locking to prevent conflicts.
   * Needed *because* multiple services are writing without coordination.

---

## Your Task

Use this schema and the column groups in **Part A** to:
1. Assign each column to its owning service (or identify it as "nobody owns it—it shouldn't exist").
2. Design what each service's schema looks like in the split model.
3. Resolve the hard cases (status, SKU, created_at, etc.).
4. Propose how reads that currently depend on this table (e.g., product detail page) will work after the split.
