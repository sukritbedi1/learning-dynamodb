# Lecture 10: Billing & Cost Optimization

## Pricing Models

### On-Demand
```
Charged per:
- Read: $0.25 per million reads
- Write: $1.25 per million writes
- Storage: $0.25 per GB-month

No capacity planning.
Scales automatically.
Great for: Variable traffic, unpredictable.
Bad for: Predictable high-volume (provisioned is cheaper).
```

### Provisioned
```
Reserve capacity:
- RCU (Read Capacity Units): 100 = 100 reads/sec of 4 KB items
- WCU (Write Capacity Units): 100 = 100 writes/sec of 1 KB items

Cost: ~$0.000013 per RCU-hour + ~$0.000065 per WCU-hour

If you overprovision: Pay but don't use (waste).
If you underprovision: Throttled (bad experience).

Great for: Predictable, high-volume (cheaper at scale).
Bad for: Unpredictable, bursty (can't handle spikes).
```

### Hybrid: AutoScaling
```
Provisioned + automatic scaling.

Start: 100 RCU
Scale up if: Utilization > 70%
Scale down if: Utilization < 30%

Benefit: Predictable baseline + burst handling.
Drawback: Takes time to scale (not instant).
```

---

## Understanding RCU/WCU

### Read Capacity Unit (RCU)
```
1 RCU = 1 strongly consistent read of 4 KB
      OR 2 eventually consistent reads of 4 KB

Example:
- Read 1 KB item: 1 RCU (even though < 4 KB)
- Read 4 KB item: 1 RCU
- Read 8 KB item: 2 RCU
- Read 10 KB item: 3 RCU (rounded up)

Query 100 items of 1 KB each:
= 100 RCU (not 25, DynamoDB reads item not byte)

Eventual consistency: 50 RCU (half cost)
```

### Write Capacity Unit (WCU)
```
1 WCU = 1 write of 1 KB

Example:
- Write 1 KB item: 1 WCU
- Write 2 KB item: 2 WCU
- Write 100 B item: 1 WCU (minimum)
- Write 5 KB item: 5 WCU

BatchWriteItem (25 items of 1 KB): 25 WCU
```

---

## Cost Estimation

### Scenario: Mid-Scale E-Commerce
```
Daily traffic:
- 1 million product views/day
- 100K orders/day
- 50K user logins/day

Reads per second (peak):
- Product views: 1M / 86400 = 11.5 reads/sec
- Order queries: 100K / 86400 = 1.2 reads/sec
- User logins: 50K / 86400 = 0.6 reads/sec
- Total: ~13 reads/sec

Writes per second (peak):
- New orders: 100K / 86400 = 1.2 writes/sec
- Inventory updates: 10K / 86400 = 0.1 writes/sec
- Total: ~1.3 writes/sec

RCU needed: 13 (or ~26 with eventual consistency)
WCU needed: 2

Provisioned Cost (monthly):
- 26 RCU × $0.000013/RCU-hour × 730 hours = $2.46
- 2 WCU × $0.000065/WCU-hour × 730 hours = $0.10
- Storage: Assume 10 GB = $2.50
Total: ~$5/month (very cost-effective)

On-Demand Cost (same volume):
- 1M reads × $0.25/M = $0.25
- 100K writes × $1.25/M = $0.125
- Storage: $2.50
Total: ~$2.88/month (even cheaper at this scale)

Takeaway: DynamoDB extremely cost-effective. Cost scale linearly with usage.
```

---

## Cost Optimization Strategies

### Strategy 1: Use Eventual Consistency
```
Strong: 1 RCU per read
Eventual: 0.5 RCU per read (50% savings)

For non-critical reads (profiles, recommendations):
Use eventual.
```

### Strategy 2: Batch Operations
```
❌ Inefficient:
for item in items {
    PutItem(item)  // 1 WCU each
}
Total: 25 WCU

✓ Efficient:
BatchWriteItem(25 items)  // Still 25 WCU but faster, atomicity
```

### Strategy 3: Project Only Needed Attributes
```
❌ Inefficient:
Query(status=pending)
→ Returns all 50 attributes

✓ Efficient:
Query(status=pending, ProjectionExpression="id,order_date,total")
→ Returns only 3 attributes
→ Smaller network transfer
→ Possible GSI with KEYS_ONLY projection
```

### Strategy 4: Filter Before Query
```
❌ Inefficient:
Query(user_id)
→ Returns 1000 items (100 RCU)
→ Filter by total > 100 (app-side)
→ Used only 10 items

✓ Efficient:
Query(user_id, total > 100)
→ DynamoDB filters (only 10 items returned)
→ ~1 RCU
```

### Strategy 5: Use On-Demand for Spiky Traffic
```
Predictable: Provisioned (cheaper)
Unpredictable spikes: On-Demand (no throttling)
Hybrid: Provisioned + AutoScaling
```

### Strategy 6: Archive Old Data
```
❌ Keep all data in DynamoDB (expensive at scale)

✓ Archive old data to S3 (cheaper)
Old orders → S3
Query recent data in DynamoDB
```

---

## Common Cost Mistakes

### Mistake 1: Overprovision
```
❌ Provision 1000 RCU, use 10
→ Paying 100x too much

✓ Start small (10 RCU), enable autoscaling
→ Scale up only if needed
```

### Mistake 2: Inefficient Queries
```
❌ Scan entire table (examines all items)
→ High RCU cost

✓ Use Query with proper keys
→ Low RCU cost
```

### Mistake 3: Hot Partition
```
❌ All queries to one partition
→ Throttled, need to overprovision

✓ Distribute queries via sharding
→ Cost-effective scaling
```

---

## Cost Monitoring

CloudWatch metrics:
- ConsumedReadCapacityUnits
- ConsumedWriteCapacityUnits
- UserErrors (throttling)
- ProvisionedReadCapacityUnits (unused capacity)

Set up alarms:
- Alert if actual > provisioned (underprovisioned)
- Alert if provisioned > actual (overprovisioned)

---

## Key Takeaway

DynamoDB cost is driven by:
1. RCU/WCU provisioned or used
2. Storage (GB-month)
3. Indexing (GSI storage + throughput)

Optimize by:
- Using eventual consistency
- Filtering at DynamoDB (not app-side)
- Provisioning correctly (or on-demand)
- Archiving old data

**Next:** [Batching & Pagination](./11-batching.md)
