# Lecture 8: Secondary Indexes (GSI, LSI)

## Why Indexes?

Primary key queries:
```
Query(PK, SK condition)
→ Fast, efficient

Non-key queries:
Query(status = "pending", total > 100)
→ Scan entire table (slow)
```

Indexes solve: "Query data differently"

---

## Global Secondary Index (GSI)

**Definition:** Alternative PK/SK structure for same data.

```
Table: Orders
PK: user_id
SK: order_id

GSI: StatusIndex
PK: status
SK: created_at

Now can query:
- By user: Query(user_id) ✓ (table)
- By status: Query(status) ✓ (GSI)
```

**How it works:**
```
DynamoDB maintains index separately.
Every item write → updates table + all GSIs
Every item delete → removes from table + all GSIs

Query(GSI: status=pending)
→ Returns order_id, user_id, total, etc.
```

### GSI Features

| Feature | Detail |
|---------|--------|
| **Separate RCU/WCU** | Index has own capacity, independent of table |
| **Eventual Consistency** | Updates not instant (eventually consistent) |
| **Sparse** | Can omit items (great for optional attributes) |
| **Size** | Can be 10GB, table 1TB (no limit) |
| **Projection** | Choose which attributes are included |

### Creating a GSI

```go
// AWS SDK example
gsi := &types.GlobalSecondaryIndex{
    IndexName: "StatusIndex",
    KeySchema: []types.KeySchemaElement{
        {AttributeName: "status", KeyType: types.KeyType("HASH")},      // PK
        {AttributeName: "created_at", KeyType: types.KeyType("RANGE")}, // SK
    },
    Projection: &types.Projection{
        ProjectionType: types.ProjectionTypeAll, // Include all attributes
    },
    ProvisionedThroughput: &types.ProvisionedThroughput{
        ReadCapacityUnits: 5,
        WriteCapacityUnits: 5,
    },
}

CreateTableInput.GlobalSecondaryIndexes = []types.GlobalSecondaryIndex{gsi}
```

### GSI Mistakes

**Mistake 1: Over-Provisioning**
```
❌ Table: 100 RCU
   StatusIndex: 100 RCU
   ProductIndex: 100 RCU
   Total: 300 RCU → $$$

✓ Start small (5 RCU each), scale based on usage
```

**Mistake 2: Including Unnecessary Attributes**
```
❌ ProjectionType: ALL
   (Every item: all 200 attributes projected)

✓ ProjectionType: KEYS_ONLY or INCLUDE(["field1", "field2"])
   (Only project what you query, reduces storage)
```

**Mistake 3: Creating Indexes You Don't Use**
```
❌ "Might need this later" index
   Sits unused, costs money, slows writes

✓ Create only indexes for documented access patterns
```

---

## Local Secondary Index (LSI)

**Definition:** Alternate SK, same PK.

```
Table: Orders
PK: user_id
SK: order_id

LSI: CreatedAtIndex
PK: user_id (same!)
SK: created_at (different)

Query(user_id, created_at > X)
→ Orders created after X, for specific user
```

**Key Difference from GSI:**

| Aspect | GSI | LSI |
|--------|-----|-----|
| **PK** | Different | Same as table |
| **Size Limit** | 10 GB per index | 10 GB total (table + all LSIs) |
| **Consistency** | Eventual | Strong (same as table) |
| **Throughput** | Separate | Shared with table |
| **Creation** | Anytime | Must define at table creation |

**Important:** LSI is rarely used. GSI is more flexible.

---

## Projection Types

### KEYS_ONLY
```
Projection: {ProjectionType: "KEYS_ONLY"}

Stores: PK + SK + Table's PK/SK
Doesn't store: Other attributes

Query returns:
{status, created_at, user_id, order_id}

Miss: Need to GetItem(table) to get total, customer_name, etc.
Cost: Smaller index; extra GetItem cost if you need more data
```

### INCLUDE
```
Projection: {ProjectionType: "INCLUDE", NonKeyAttributes: ["total", "customer_name"]}

Stores: Keys + specified attributes
Other attributes: Not included

Query returns:
{status, created_at, user_id, order_id, total, customer_name}

No extra GetItem needed for these attributes.
Cost: Larger index; faster queries
```

### ALL
```
Projection: {ProjectionType: "ALL"}

Stores: All attributes
Cost: Largest index size; fastest queries (no extra GetItem)
```

**Rule:** Use KEYS_ONLY + occasional GetItem, or INCLUDE if you query specific attributes often.

---

## GSI Use Cases

### Case 1: Reverse Lookup
```
Table: Users
PK: user_id

GSI: EmailIndex
PK: email
SK: user_id

Query(email="alice@example.com")
→ Find user by email (for login)
```

### Case 2: Time-Series Alternate View
```
Table: Metrics
PK: metric_name
SK: timestamp

GSI: HostMetrics
PK: host_id
SK: timestamp

Query(host_id, timestamp > X)
→ Get all metrics for host over time
```

### Case 3: Multi-Tenant
```
Table: Data
PK: entity_id

GSI: ByTenant
PK: tenant_id
SK: entity_id

Query(tenant_id)
→ Get all entities for tenant (if original table doesn't support it)
```

---

## Sparse Indexes (Optimization)

```
Table: Users
GSI: PremiumIndex
PK: subscription_type
SK: user_id

Items:
{user_id: 1, subscription_type: "premium"}     → In GSI
{user_id: 2, subscription_type: null}          → NOT in GSI
{user_id: 3}                                    → NOT in GSI

Query(subscription_type="premium")
→ Only returns premium users (filtered automatically)

Benefit: Index stores only relevant items, smaller.
```

---

## Key Takeaway

**GSI:** Flexible, anytime, different PK. Use for alternate access patterns.
**LSI:** Rare, same PK, strong consistency. Use when you need strong consistency + sort key.

Indexes enable: Query data differently.

Start with primary key queries. Add GSI only for documented access patterns.

**Next:** [Read/Write Consistency](./09-consistency.md)
