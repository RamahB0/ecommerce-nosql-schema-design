# Part 2: Refactor for Analytics and High Availability

## 1. What changed

Two new requirements are added on top of Part 1. Analytics means large-scale analytical queries over product trends and sales data, such as units sold per product per day, top categories this month, or revenue trend over ninety days. High availability means the system must tolerate node and network-partition failures gracefully as the user base grows, which is to say it must lean toward the A and P of CAP for the parts of the system where that trade-off is acceptable.

Neither requirement is well served by the Part 1 schema as-is. Running analytical aggregations directly against orders_by_id or orders_by_customer would mean full scans across a table that is deliberately optimized for high-throughput single-partition writes; every analytical query would compete with live checkout traffic for the same nodes. And Part 1 said nothing about replica placement or partition counts, which is what actually determines availability under node or network failure.

## 2. Strategy: sharding and replication for availability, denormalization for analytics

### 2.1 Sharding

Both stores are explicitly partitioned across multiple nodes, with shard keys chosen to avoid hotspots. The products collection is sharded on a hashed SKU, not on category.id; sharding by category would concentrate all "Electronics" writes and reads on one shard, since a handful of categories dominate traffic, while hashing the SKU spreads any single category evenly across shards. orders_by_id is already partitioned by order_id (Part 1); at higher volume this becomes a hashed partition key rather than a raw sequential ID, for the same reason: a monotonically increasing key would concentrate the newest, hottest writes on a single partition or node. orders_by_customer is partitioned by customer_id, which is naturally well distributed across a large user base.

### 2.2 Replication

Each shard is replicated across at least three nodes spread over separate availability zones, and, for the order store, across multiple regions. The document store runs as replica sets per shard, one primary and two or more secondaries. Catalog reads use a secondary-preferred read preference, so read traffic is served even if a primary is unreachable, and a primary failover does not take catalog browsing down with it. The wide-column order store uses multi-datacenter replication, replication factor three per datacenter. Writes and the status-critical reads from Part 1 use a quorum consistency level, a majority of replicas must acknowledge, which keeps delivery_status and payment.status correct even though the system as a whole is tuned for availability; less critical reads, such as populating an order history list, can use a weaker, single-replica consistency level for lower latency.

This is the concrete CAP trade-off: under a network partition, the order store keeps accepting writes on the majority side (available, partition-tolerant) and briefly refuses or delays writes on a minority side rather than risking two conflicting "delivered" statuses for the same order. Availability is sacrificed only for the specific fields where correctness matters, not for the whole write path.

### 2.3 Denormalization for analytics

Analytics is solved by not querying the operational stores at all. Every order write and every catalog update is captured as a change event (change streams from the document store, change data capture from the wide-column store) and streamed into a separate, denormalized analytics layer. This is a CQRS-style split: one schema optimized for writing transactions, a different schema, fed asynchronously, optimized for reading aggregates.

Two new analytics-facing tables are added. daily_product_sales holds one row per product per day, incrementally updated as order events arrive, answering "units sold or revenue for product X over date range Y" without touching a single live order record:

```
{"product_id": "PROD-1001", "date": "2026-08-22", "units_sold": 134, "revenue": 10718.66, "category_id": "C10"}
```

category_trends is a rolling window rollup, recomputed on a schedule or incrementally via a streaming aggregation job, answering "what's trending" without a scan over raw orders:

```
{"category_id": "C10", "window": "7d", "as_of": "2026-08-23", "units_sold": 9820, "revenue": 784112.40, "trend_vs_prior_window_pct": 6.4}
```

These aggregate tables are intentionally eventually consistent with, and slightly behind, the operational stores, typically seconds to a few minutes of lag from the streaming pipeline. That lag is acceptable for a "what are this week's top sellers" dashboard and unacceptable for "did this specific order get paid," which is exactly why the two concerns are kept in separate stores with separate consistency guarantees rather than one schema trying to serve both.

## 3. Trade-offs

Sharding and multi-region replication buy availability and write scalability at the cost of operational complexity: shard-key choice becomes a first-class design decision, since a bad key creates a hotspot no amount of hardware fixes, and cross-shard queries, such as total revenue across all products if it were run live, become expensive or impossible without the analytics layer. Denormalizing into daily_product_sales and category_trends buys fast analytical reads and takes the analytics workload off the transactional path entirely, at the cost of storage duplication and staleness, since the numbers on a trends dashboard are always slightly in the past. Favoring availability for catalog reads and for most of the order record means the system can present slightly stale product data during a partition rather than going down, while the narrower quorum-write requirement on order and payment status accepts some latency and, in a severe partition, temporary write unavailability on the minority side, in exchange for never recording two contradictory outcomes for the same order.
