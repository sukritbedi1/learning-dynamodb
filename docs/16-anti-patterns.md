# Lecture 16: Common Pitfalls & Anti-Patterns

## Anti-Pattern 1: Query Everything

❌ WRONG:
```
Query(filter: total > 100 AND status = "pending" AND region = "us-east")
→ Scans entire table
→ Returns 10 items from 1M items examined
→ Expensive
```

✓ CORRECT:
```
Design sort key to support: Query(pk, sk has status AND region)
→ Query directly returns 10 items
→ No scan needed
```

**Lesson:** Use indexes to narrow query scope.

---

## Anti-Pattern 2: Hot Partitions

❌ WRONG:
```
PK: "GLOBAL_COUNTER"
Every order → increment counter

Result: All writes to one partition → throttled
```

✓ CORRECT:
```
PK: "COUNTER#shard#{0..9}"
Random shard per write

Result: Writes distributed → no throttling
```

**Lesson:** Distribute partition key for even load.

---

## Anti-Pattern 3: Storing Blobs in DynamoDB

❌ WRONG:
```
{
    id: "image#123",
    image_data: "<10 MB PNG blob>"  // Item size limit: 400 KB
}
→ Fails
```

✓ CORRECT:
```
{
    id: "image#123",
    s3_url: "s3://bucket/images/123.png"  // Reference only
}
→ Upload blob to S3
→ Store URL in DynamoDB
```

**Lesson:** Large files → S3, reference in DynamoDB.

---

## Anti-Pattern 4: Overusing Transactions

❌ WRONG:
```
For every single operation, wrap in transaction.
→ Higher latency, higher cost
→ No actual need for atomicity
```

✓ CORRECT:
```
Single item write: No transaction needed.
Related items (order + inventory): Use transaction.
```

**Lesson:** Transactions have overhead. Use only when atomicity needed.

---

## Anti-Pattern 5: Using Scan for Business Logic

❌ WRONG:
```
Get all pending orders:
Scan(filter: status = "pending")
→ Examines 1M items, returns 10K

Every application startup, every query: Scan!
```

✓ CORRECT:
```
GSI: status as PK
Query(status="pending")
→ Returns 10K items directly
```

**Lesson:** Scan is for backups/exports. Query for business logic.

---

## Anti-Pattern 6: No Pagination

❌ WRONG:
```
Query(user_id)
→ Returns 1M items (1 GB)
→ Crashes application

No Limit parameter.
```

✓ CORRECT:
```
Query(user_id, Limit=100)
→ Returns 100 items + pagination token
→ App handles pagination
```

**Lesson:** Always paginate for large result sets.

---

## Anti-Pattern 7: Ignoring Eventual Consistency

❌ WRONG:
```
Write data.
Immediately read (eventual consistency).
→ See old data
→ User confused

Not aware of eventual consistency implications.
```

✓ CORRECT:
```
Write data.
For critical reads (inventory): Strong consistency.
For non-critical (profile): Eventual consistency.
→ Be aware, handle appropriately in app.
```

**Lesson:** Understand eventual consistency trade-offs.

---

## Anti-Pattern 8: Wrong Data Type

❌ WRONG:
```
"age": "30"  // String, not Number
"created_at": "2024-09-27"  // String, not Number

Query(age > 30)
→ Returns "4", "5", "9" (lexicographic sort)
→ Wrong results
```

✓ CORRECT:
```
"age": 30  // Number
"created_at": 1695813000  // Unix timestamp (Number)

Query(age > 30)
→ Returns 31, 40, 100 (numeric sort)
```

**Lesson:** Choose correct types for queries.

---

## Anti-Pattern 9: Missing Indexes

❌ WRONG:
```
"We might query by email later."
Don't create index.

Later: Query(email="alice@example.com")
→ Scan entire table
→ Slow
```

✓ CORRECT:
```
Know access patterns upfront.
Create email GSI at table creation.
→ Query(email) fast
```

**Lesson:** Plan indexes before launch.

---

## Anti-Pattern 10: Tight Provisioning

❌ WRONG:
```
Provision exactly what you need:
Measured traffic: 100 RCU
Provision: 100 RCU

Spike: 150 RCU
→ Throttled
```

✓ CORRECT:
```
Provision with headroom or use autoscaling:
Measured: 100 RCU
Provision: 150 RCU (50% headroom)
→ Handle spikes gracefully

Or: Use on-demand (no provisioning).
```

**Lesson:** Plan for spikes or use on-demand.

---

## DynamoDB Success Checklist

- [ ] Define access patterns first
- [ ] Choose partition key for even distribution
- [ ] Design sort key to support queries
- [ ] Use GSI for alternate patterns (not Scans)
- [ ] Plan indexes before launch
- [ ] Avoid blobs (use S3)
- [ ] Paginate large results
- [ ] Monitor capacity utilization
- [ ] Set up alarms for throttling
- [ ] Understand eventual consistency
- [ ] Use correct data types
- [ ] Batch operations for efficiency
- [ ] Use transactions only when needed
- [ ] Document your schema design

---

## Key Takeaway

Common mistakes:
1. Querying without indexes (Scans)
2. Hot partitions (poor key design)
3. Blobs in DynamoDB (use S3)
4. Ignoring consistency model
5. Wrong data types

Avoid them by:
- Planning access patterns upfront
- Distributing keys evenly
- Using indexes strategically
- Understanding trade-offs

---

## Where to Go From Here

**Next Step:** Implement with Go SDK.

**Topics Not Covered (Advanced):**
- Encryption at rest/in transit
- Fine-grained access control (IAM)
- DynamoDB Accelerator (DAX) caching
- Backup & restore strategies
- Reserved capacity (enterprise)
- On-demand backup

**Resources:**
- AWS DynamoDB Documentation
- AWS Best Practices Guide
- Real-world case studies

---

## Session Complete

You've learned:
1. ✓ Fundamentals (what, why, when to use)
2. ✓ Mental model (RDS vs DynamoDB)
3. ✓ Architecture (keys, partitions, indexes)
4. ✓ Operations (queries, batching, pagination)
5. ✓ Advanced (transactions, streams, global tables)
6. ✓ Production (monitoring, billing, anti-patterns)

**Ready for:** Go SDK implementation, building real services.

---

**Go forth and build fast, scalable systems.** 🚀
