# E-Commerce NoSQL Database Design

This repository contains a two-part NoSQL schema design exercise for an e-commerce application: a product catalog with search, an order pipeline that must sustain thousands of transactions per second, and a refactor that adds large-scale analytics and high availability on top of the original design.

## Contents

Part 1, the initial schema design (entity model, indexing, and consistency choices), is in docs/01-initial-schema.md. Part 2, the refactor for analytics and high availability (sharding, replication, denormalization), is in docs/02-refactored-schema.md. The short reflection report (200-300 words) on the refactor is in docs/03-reflection.md.

## Summary

The design uses a polyglot NoSQL architecture rather than a single database technology, since the catalog and the order pipeline have very different access patterns.

Product catalog and user profiles are stored in a document database (MongoDB-style), because product attributes vary by category, catalog reads dominate, and full-text search is required. Orders are stored in a wide-column / key-value store (Cassandra/DynamoDB-style), because order writes are the highest-throughput, most latency-sensitive path in the system and benefit from partition-based horizontal scaling and tunable consistency.

The refactor in Part 2 layers sharding and multi-region replication onto both stores for availability, and introduces a separate, denormalized analytics store fed by change-data-capture so analytical queries never compete with the operational write path.
