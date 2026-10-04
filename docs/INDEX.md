# DynamoDB: Lecture Notes

Comprehensive notes on DynamoDB — from first principles to production patterns.

## Lecture Series

### Module 1: Fundamentals
1. [What is DynamoDB?](./01-what-is-dynamodb.md) — Overview, use cases, when to use
2. [RDBMS vs NoSQL: Mental Model Shift](./02-rdbms-vs-nosql.md) — Key differences from MySQL/Postgres
3. [Core Concepts](./03-core-concepts.md) — Tables, items, attributes, keys, partitions
4. [Data Types & Schema](./04-data-types.md) — Scalar, document, set types; schema design

### Module 2: Architecture & Design
5. [Partition Keys & Distribution](./05-partition-keys.md) — Hot keys, sharding strategies
6. [Sort Keys & Queries](./06-sort-keys.md) — Range queries, begins_with, between
7. [Access Patterns](./07-access-patterns.md) — Design-first approach; query patterns before schema
8. [Secondary Indexes](./08-secondary-indexes.md) — GSI, LSI, use cases, cost trade-offs

### Module 3: Operations & Performance
9. [Read/Write Consistency](./09-consistency.md) — Strong vs eventual, RCU/WCU model
10. [Billing & Cost Optimization](./10-billing.md) — On-demand vs provisioned, burst capacity, scaling
11. [Batching & Pagination](./11-batching.md) — BatchGetItem, BatchWriteItem, Query limits
12. [Transactions](./12-transactions.md) — ACID guarantees, TransactGetItems, TransactWriteItems

### Module 4: Advanced Topics
13. [Streams & Change Data Capture](./13-streams.md) — Lambda triggers, event processing
14. [Global Tables & Replication](./14-global-tables.md) — Multi-region, conflict resolution
15. [Monitoring & Observability](./15-monitoring.md) — CloudWatch metrics, throttling, alarms
16. [Common Pitfalls & Anti-Patterns](./16-anti-patterns.md) — What breaks, how to fix

---

## Quick Reference

| RDBMS Concept | DynamoDB Equivalent | Key Difference |
|---|---|---|
| Database | DynamoDB (service) | Serverless, no instances |
| Table | Table | No schema enforcement |
| Row | Item | Flexible attributes |
| Column | Attribute | Type: S/N/B/SS/NS/BS/M/L |
| Primary Key | Partition Key + Sort Key | Must plan access patterns first |
| Index | GSI/LSI | Queries design access patterns |
| Join | Denormalization | No joins; duplicate data |
| Relationship | Item references | Manual, app-side joins |
| Transaction | TransactWrite | Limited scope, 25 items max |
| Scaling | Auto-scale RCU/WCU | Pay-per-request or provisioned |

---

**Study Approach:** Read in order. Each lecture builds on prior.
When ready: implement examples in Go SDK.

---

Last Updated: 2026-09-27
