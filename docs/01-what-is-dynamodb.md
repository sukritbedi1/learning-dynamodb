# Lecture 1: What is DynamoDB?

## Definition

**DynamoDB** is AWS's managed, serverless NoSQL database service.

"Managed" = AWS handles: infrastructure, scaling, backups, patches, failover.
"Serverless" = You don't provision capacity; you pay per request or reserve capacity.
"NoSQL" = Flexible schema, optimized for specific access patterns, not relational queries.

## High-Level Architecture

```
Your Application
    ↓
AWS SDK (Go, Python, Node, etc.)
    ↓
DynamoDB API (HTTP/JSON)
    ↓
DynamoDB Service (Serverless, managed)
    ├── Partition 1: Items A-M
    ├── Partition 2: Items N-Z
    └── Replicas (multi-region, optional)
    ↓
Persistent Storage (SSD, encrypted)
```

You interact via **SDK calls**. AWS handles everything else.

## Key Characteristics

| Aspect | Details |
|--------|---------|
| **Storage Model** | Key-Value + sorting; scalable to any size |
| **Schema** | No schema enforcement; items in same table can differ |
| **Queries** | Simple (get by key), range (key prefix/range), scans (inefficient) |
| **Joins** | Not supported; must denormalize or app-side join |
| **Transactions** | Limited ACID (up to 25 items, per-item ≤ 4 MB) |
| **Replication** | Single-region (default) or multi-region (Global Tables) |
| **Consistency** | Strong (for single-item) or Eventual (cheaper, slightly stale) |
| **Scaling** | Automatic; pay what you use (on-demand) or reserve (provisioned) |

## When to Use DynamoDB

✅ **Good fit:**
- Real-time data (user profiles, sessions, game state)
- High throughput (100k+ writes/sec)
- Uniform access patterns (same queries across data)
- Flexible schema (rapidly changing attributes)
- Serverless (no ops burden)
- Mobile apps, IoT, live feeds

❌ **Poor fit:**
- Complex joins across many tables
- Ad-hoc analytics (use Redshift, Athena instead)
- OLTP reporting (use RDS instead)
- Strong schema enforcement needed
- Transactions across 25+ items
- SQL queries (use RDS/Aurora)

## DynamoDB vs Relational Databases

| Aspect | RDS (MySQL/Postgres) | DynamoDB |
|--------|----------------------|----------|
| **Schema** | Strict; define upfront | Flexible; add attributes anytime |
| **Queries** | SQL; query any combination | Access patterns; must plan first |
| **Joins** | Native (JOIN clause) | Not supported; denormalize |
| **Indexes** | Can add indexes later | Must plan GSI/LSI upfront |
| **Scaling** | Vertical (bigger instances); hard limits | Horizontal (partitions); unlimited |
| **Cost** | Fixed (instance running 24/7) | Variable (pay per request or capacity) |
| **Consistency** | ACID by default | Eventual by default; strong available |
| **Transactions** | Full ACID across tables | Limited (25 items, 4 MB max) |
| **Ops** | Manage backups, patches, replication | AWS manages everything |
| **Learning Curve** | SQL; familiar for DBAs | Access patterns; mental shift required |

## Real-World Analogy

**RDS** is like a library with a card catalog. You can search any way (by author, title, subject). Flexible but requires a librarian (you) to maintain indexes.

**DynamoDB** is like a vending machine. You know exactly what button to press (access pattern). Machine is always stocked and fast. But you can't ask "give me all red items" — you have to know the button upfront.

## Deployment Models

### Local Development: DynamoDB Local / Ministack
- Run on your machine (Docker)
- Same API as AWS
- No charges
- Good for testing

### Production: AWS
- Single-region (default)
- Multi-region (Global Tables)
- On-demand or provisioned capacity
- Automatic scaling
- CloudWatch monitoring

---

## Key Takeaway

DynamoDB trades **query flexibility** for **performance and scale**. You must think in access patterns, not ad-hoc queries. Once you adopt this mindset, DynamoDB is incredibly powerful.

**Next:** [RDBMS vs NoSQL: Mental Model Shift](./02-rdbms-vs-nosql.md)
