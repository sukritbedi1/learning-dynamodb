# Lecture 5: Partition Keys & Distribution

## Partition Key Design Goals

1. **Even distribution:** Queries spread across partitions
2. **No hot keys:** No single partition gets hammered
3. **Query efficiency:** Support your access patterns

---

## Choosing a Partition Key

### Criteria

**High Cardinality** (many unique values)
```
❌ BAD: is_active (only true/false → 2 partitions)
✓ GOOD: user_id (millions unique → millions partitions)
```

**Uniform Distribution** (evenly distributed queries)
```
❌ BAD: country (90% queries US → US partition overloaded)
✓ GOOD: user_id (requests spread across users)
```

**Access Pattern Match** (supports how you query)
```
❌ BAD: product_id when you always query by user
        (Have to scan all product partitions)

✓ GOOD: user_id when queries are "get user + their data"
        (One partition, one query)
```

---

## Common Partition Key Patterns

### Pattern 1: Entity ID
```
Table: Users
PK: user_id

GetItem(PK="user#123")
→ One partition
→ Fast
→ Works if all data for user is in one item
```

**Pro:** Simple, direct.
**Con:** Data duplication if you have related items (orders, posts).

### Pattern 2: Entity ID + Type (Single Table)
```
Table: UserData
PK: user_id
SK: type

Items:
{pk: user#123, sk: PROFILE, ...}
{pk: user#123, sk: ORDER#456, ...}
{pk: user#123, sk: POSTS, ...}

Query(PK="user#123")
→ All data for user in one partition
→ One query gets profile + orders + posts
```

**Pro:** Efficient single queries; related data together.
**Con:** Complex schema; key design critical.

### Pattern 3: Tenant ID (Multi-Tenant)
```
Table: TenantData
PK: tenant_id
SK: entity_id

Items:
{pk: tenant#acme, sk: order#456, ...}
{pk: tenant#acme, sk: user#123, ...}
{pk: tenant#globex, sk: order#789, ...}

Query(PK="tenant#acme")
→ All data for one tenant

Isolation:
Each tenant's data isolated in partition
Queries don't cross tenants
```

**Pro:** Automatic data isolation; multi-tenancy built-in.
**Con:** All queries must know tenant.

### Pattern 4: Sharded ID (Handling Hot Keys)
```
When: Single entity queried very frequently

Solution: Shard the partition key

Instead of:
PK: "USER#hot_user"  ← All queries to one partition

Use:
PK: "USER#hot_user#SHARD#0"
PK: "USER#hot_user#SHARD#1"
PK: "USER#hot_user#SHARD#2"
PK: "USER#hot_user#SHARD#3"

Then:
Queries round-robin across shards
Load distributed
```

**Pro:** Handles hot keys at query time.
**Con:** Queries need to know shard count; app-side logic.

Example code:
```go
// Determine shard (0-3)
shard := rand.Intn(4)
pk := fmt.Sprintf("USER#hot_user#SHARD#%d", shard)

// Query shard
response := GetItem(pk, sk)
```

### Pattern 5: Date/Region Sharding
```
Table: Metrics
PK: "REGION#AWS#DATE#2024-09-27"
SK: timestamp

Items:
{pk: REGION#AWS#DATE#2024-09-27, sk: 1695813000, value: 99.5}
{pk: REGION#AWS#DATE#2024-09-27, sk: 1695813060, value: 99.2}
{pk: REGION#GCP#DATE#2024-09-27, sk: 1695813000, value: 87.3}
{pk: REGION#GCP#DATE#2024-09-28, sk: 1695813000, value: 88.1}

Query(PK="REGION#AWS#DATE#2024-09-27")
→ All AWS metrics for one day
→ One partition
→ Good for time-series
```

**Pro:** Natural data organization; date-based rotation.
**Con:** Need to know date when querying old data.

---

## Partition Key Mistakes

### Mistake 1: Low Cardinality
```
❌ WRONG:
PK: status ("active" or "inactive")

Results:
Active status → partition A (millions of users)
Inactive status → partition B (few users)

Partition A: Throttled
Partition B: Idle
```

### Mistake 2: Skewed Distribution
```
❌ WRONG:
PK: country

Results:
US: 90% of queries → partition overloaded
India: 5% of queries → partition underloaded
Japan: 3% of queries → partition underloaded
```

### Mistake 3: Query Mismatch
```
❌ WRONG:
PK: product_id
But most queries: "Get all orders by user"

Results:
Query(PK="product#milk")
→ Find one partition
→ Need all orders → scan entire table
→ Inefficient
```

---

## Partition Key Evolution

**Can you change partition key after launch?**
No. Immutable.

**What if you choose wrong?**
Options:
1. Create new table with correct PK
2. Migrate data (scan old, write to new)
3. Use GSI as workaround (next lecture)

**Prevention:**
Plan PK carefully before launch. It's hard to change.

---

## Monitoring Partition Distribution

CloudWatch metrics:
- **ConsumedReadCapacityUnits per Partition Key:** Should be even
- **UserErrors:** High = hot key throttling
- **Latency:** Should be consistent for same partition

If uneven:
- Repartition (create new table)
- Add sharding (Lecture 5 pattern)
- Use GSI with better key (next lecture)

---

## Key Takeaway

Partition key is the most critical design decision.

**Good PK:**
- High cardinality (many unique values)
- Uniform distribution (evenly distributed queries)
- Matches access patterns (supports how you query)

**Bad PK:** Low cardinality, skewed distribution, or query mismatch.

Choose carefully. Changing later is expensive.

**Next:** [Sort Keys & Queries](./06-sort-keys.md)
