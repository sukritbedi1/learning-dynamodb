# Lecture 14: Global Tables & Replication

## Global Tables

**Definition:** Managed multi-region replication of DynamoDB table.

```
Region US-East-1:
Table: Orders
└── Replicated to...

Region EU-West-1:
Table: Orders (replica)
└── Bidirectional sync

Region AP-South-1:
Table: Orders (replica)
```

---

## Features

| Feature | Detail |
|---------|--------|
| **Active-Active** | Write to any region, all updated |
| **Eventual Consistency** | Replication within milliseconds |
| **No App Changes** | Use same SDK, tables appear local |
| **Auto Failover** | If region fails, read/write to another |
| **Global Indexes** | Separate RCU/WCU per region |

---

## When to Use

```
Use:
- Serve globally (low latency for users worldwide)
- Disaster recovery (region fails, still available)
- Compliance (data residency: keep data in-region)

Don't:
- Single region app (complexity not needed)
- Strong consistency requirements (eventual only)
```

---

## Setup

```go
// Create table in primary region
// Then create replica in secondary

GlobalSecondaryIndexes: {
    {
        IndexName: "Orders",
        Replicas: [
            {RegionName: "us-east-1"},
            {RegionName: "eu-west-1"},
        ],
    },
}
```

---

## Replication Latency

```
Write in US → Replicate to EU
Latency: Usually <1 second
Eventual consistency: Changes appear in EU shortly

Read from EU immediately after US write:
Might see old data (briefly)

Eventual consistency always.
```

---

## Key Takeaway

Global Tables: Multi-region active-active replication.

Enables global low-latency access + disaster recovery.

Trade: Eventual consistency, complexity.

**Next:** [Monitoring & Observability](./15-monitoring.md)
