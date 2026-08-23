# Part 1: Initial Schema Design

## 1. Key entities

The core domain has three entities plus one supporting entity. User is the account, profile, and saved addresses. Product is the catalog item, with category, price, attributes, and inventory. Order is a purchase made by a user, containing a snapshot of the items bought, shipping and billing details, payment status, and delivery status. Category is treated as a supporting entity, embedded as a field on Product rather than modeled as its own collection, since category browsing is really "filter products by category path," not a heavyweight entity with its own lifecycle.

## 2. Choice of NoSQL model

A single database technology does not fit both workloads well, so this design is polyglot: two NoSQL models, chosen per access pattern rather than per entity count.

Product catalog and users go into a document database (MongoDB-style). Products have highly variable attributes per category (a headphone has "battery life," a t-shirt has "size" and "color"), which a rigid relational schema handles poorly but a document's flexible, per-record structure handles naturally. Catalog browsing is read-heavy and benefits from secondary and text indexes that document databases provide out of the box.

Orders go into a wide-column / key-value store (Cassandra/DynamoDB-style). The requirement of "thousands of transactions per second" is really a requirement on the order-write path, not the catalog. Wide-column stores are built for exactly this: horizontally partitioned, append-friendly writes with a predictable, tunable consistency model, at the cost of not supporting ad-hoc secondary queries, which is fine, because order access patterns are known in advance (by order ID, by customer).

This is a deliberate trade-off: a single document database for everything would be simpler operationally, but would force one consistency and throughput profile onto two workloads that need different ones.

## 3. Schema definitions

### 3.1 Users (document store, collection users)

A user document holds the account profile and a small, bounded array of saved addresses.

```
{"_id": "U1001", "email": "jane.doe@example.com", "password_hash": "bcrypt$...", "first_name": "Jane", "last_name": "Doe", "phone": "+212600000000", "addresses": [{"address_id": "A1", "type": "shipping", "line1": "12 Rue Atlas", "city": "Casablanca", "zip": "20000", "country": "MA", "is_default": true}], "created_at": "2025-11-02T09:00:00Z", "updated_at": "2026-08-01T14:22:00Z"}
```

Addresses are embedded because a user has a handful of addresses, not thousands, which is the classic case where embedding beats referencing in a document model: the whole profile is fetched in one read, with no join.

### 3.2 Products (document store, collection products)

```
{"_id": "PROD-1001", "name": "Wireless Headphones", "description": "Over-ear Bluetooth headphones with active noise cancellation.", "category": {"id": "C10", "path": "Electronics > Audio > Headphones"}, "brand": "AudioTech", "price": {"amount": 79.99, "currency": "USD"}, "attributes": {"color": "black", "wireless": true, "battery_life_hrs": 20}, "inventory": {"stock_qty": 542, "warehouse_locations": ["WH-EAST", "WH-WEST"]}, "ratings": {"avg": 4.5, "count": 1023}, "search_keywords": ["headphones", "wireless", "bluetooth", "noise cancelling"], "created_at": "2025-06-01T08:00:00Z", "updated_at": "2026-08-20T11:05:00Z"}
```

The attributes field is an open, schema-less sub-document by design: different categories populate different keys, and the application does not need a migration every time a new product type is added.

### 3.3 Orders (wide-column store)

Order access has exactly two known patterns: fetch one order by ID (checkout confirmation, support lookups, webhooks), and fetch a customer's order history, most recent first. A relational or single-document model would answer both from one table via a query; a wide-column store instead requires query-first modeling, a denormalized table per access pattern, both written together.

Table orders_by_id, partition key order_id, is the canonical record:

```
{"order_id": "O20260823-0001", "customer_id": "U1001", "customer_snapshot": {"name": "Jane Doe", "email": "jane.doe@example.com"}, "items": [{"sku": "PROD-1001", "name": "Wireless Headphones", "qty": 2, "unit_price": 79.99}], "shipping_address": {"line1": "12 Rue Atlas", "city": "Casablanca", "zip": "20000", "country": "MA"}, "payment": {"method": "credit_card", "status": "captured", "transaction_id": "TXN-88291"}, "delivery_status": "shipped", "totals": {"subtotal": 159.98, "tax": 12.80, "shipping": 5.00, "total": 177.78}, "created_at": "2026-08-20T10:15:00Z", "updated_at": "2026-08-21T09:00:00Z"}
```

Table orders_by_customer, partition key customer_id, clustering key order_date descending then order_id, is a slimmer projection used only to page through a customer's history without scanning the whole orders table:

```
{"customer_id": "U1001", "order_date": "2026-08-20T10:15:00Z", "order_id": "O20260823-0001", "total": 177.78, "delivery_status": "shipped"}
```

Both customer_snapshot and each line item's name and unit_price are copied at order time, not referenced. This is intentional: an order is a legal and financial record of what was actually purchased, and it must stay correct even after the customer changes their name or a product's price or listing changes later. Referencing live users/products documents would silently corrupt order history.

## 4. Relationships and indexes

Relationships are handled by embedding where the child is small and owned by the parent (addresses in a user, line items in an order), and by denormalized copies where the two entities have independent lifecycles but the order must be immutable (customer and product snapshots in an order). There are no foreign-key joins; every read is designed to be satisfiable from a single collection or table.

On products, a unique index on the SKU supports direct lookups, a compound index on category.id plus price.amount supports category browsing with price sort and filter, and a text index over name, description, and search_keywords supports full-text search. At real scale this text index is offloaded to a dedicated search engine (Elasticsearch or OpenSearch, or a managed search index) kept in sync via change streams, since native document-database text search does not scale to catalog-wide relevance ranking and typo tolerance. On users, a unique index on email supports login lookups. On orders_by_id, the partition key order_id is the primary and only lookup key; this table is not queried any other way. On orders_by_customer, the partition key customer_id plus clustering key order_date gives "most recent orders first" for free from the storage layout, with no in-memory sort.

## 5. Scalability and consistency

The two stores intentionally sit at different points on the consistency spectrum.

Product catalog favors availability and read throughput over strict consistency. A product's price or stock count being a few seconds stale on a replica is an acceptable cost for spreading catalog reads across many nodes; writes (price and inventory updates) are comparatively rare next to the read volume.

Orders need stronger guarantees on specific fields. delivery_status and payment.status are updated by concurrent processes (fulfillment service, payment webhook, customer support), so writes to those fields use a conditional, compare-and-set update (a lightweight transaction in Cassandra terms, a conditional write in DynamoDB terms) to prevent one process silently overwriting another's status change. Everything else about the order is written once at creation and never mutated, which keeps the hot write path, inserting a new order, a simple, fast, single-partition append that scales linearly by adding nodes and choosing a partition key (order_id, itself derived from a hash) that spreads load evenly and avoids the classic "last few hours of orders all land on one node" hotspot.
