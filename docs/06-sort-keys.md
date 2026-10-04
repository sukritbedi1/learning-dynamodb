# Lecture 6: Sort Keys & Queries

## Sort Key Purpose

**Partition Key:** Which partition?
**Sort Key:** Order within partition + enables range queries.

```
Partition for user#123:
├── SK: "ORDER#001" → first order
├── SK: "ORDER#002" → second order
├── SK: "ORDER#003" → third order
└── SK: "ORDER#999" → last order

Sorted by SK value (lexicographic or numeric).
```

---

## Sort Key Benefits

### 1. Ordering
```
Table: Orders
PK: user_id
SK: created_at (timestamp)

Items sorted by creation time within partition:
Query(PK="user#123")
→ Returns orders chronologically (oldest first)
```

### 2. Range Queries
```
❌ Without sort key:
Query(PK="user#123")
→ Get all orders (no filtering)

✓ With sort key:
Query(PK="user#123", SK > 1695000000)
→ Get orders after timestamp (range query!)
```

### 3. Prefix Matching
```
Table: UserPosts
PK: user_id
SK: post_id (format: "YEAR#2024#MONTH#09#DAY#27#ID#abc123")

Query(PK="user#123", SK begins_with "YEAR#2024#MONTH#09")
→ Get all posts in September 2024 (prefix match!)
```

---

## Sort Key Operations

### Exact Match
```
Query(
  PK = "user#123",
  SK = "ORDER#456"
)
→ Returns one item (if exists)
```

### Greater Than / Less Than
```
Query(
  PK = "user#123",
  SK > 1695000000
)
→ All orders after timestamp

Query(
  PK = "user#123",
  SK <= 1695000000
)
→ All orders up to timestamp
```

### Between
```
Query(
  PK = "user#123",
  SK between 1695000000 and 1695100000
)
→ All orders in time range
```

### Begins With (Prefix)
```
Query(
  PK = "user#123",
  SK begins_with "ORDER#"
)
→ All items starting with "ORDER#" (all orders)

Query(
  PK = "user#123",
  SK begins_with "POST#2024#09"
)
→ All posts in September 2024
```

### Reverse Order
```
Query(
  PK = "user#123",
  SK between low and high,
  ScanIndexForward = false  // Descending
)
→ Items in reverse order (newest first)
```

---

## Sort Key Design Patterns

### Pattern 1: Timestamp (Time Series)
```
Table: Metrics
PK: metric_name
SK: timestamp

Items:
{pk: cpu_usage, sk: 1695813000, value: 95.5}
{pk: cpu_usage, sk: 1695813060, value: 94.2}
{pk: cpu_usage, sk: 1695813120, value: 96.1}

Query(PK="cpu_usage", SK > 1695813000)
→ Get metrics after time (monitoring, charts)
```

**Use case:** Time-series data, logs, metrics, events.

### Pattern 2: Hierarchical (Composite SK)
```
Table: Inventory
PK: store_id
SK: "CATEGORY#Food#PRODUCT#Milk"

Items:
{pk: store#NYC, sk: CATEGORY#Food#PRODUCT#Milk}
{pk: store#NYC, sk: CATEGORY#Food#PRODUCT#Bread}
{pk: store#NYC, sk: CATEGORY#Drinks#PRODUCT#Coffee}

Query(PK="store#NYC", SK begins_with "CATEGORY#Food")
→ All food items in store

Query(PK="store#NYC", SK begins_with "CATEGORY#Food#PRODUCT#")
→ Just products (not categories)
```

**Use case:** Categories, hierarchies, nested data.

### Pattern 3: Status + Timestamp
```
Table: Orders
PK: user_id
SK: "STATUS#pending#created_at#1695813000"

Items:
{pk: user#123, sk: STATUS#pending#created_at#1695813000}
{pk: user#123, sk: STATUS#shipped#created_at#1695813100}
{pk: user#123, sk: STATUS#delivered#created_at#1695813200}

Query(PK="user#123", SK begins_with "STATUS#pending")
→ All pending orders

Query(PK="user#123", SK between "STATUS#pending#..." and "STATUS#shipped#...")
→ Orders between statuses (range)
```

**Use case:** Status tracking, state transitions.

### Pattern 4: Reverse Lookups (Avoid Scans)
```
Table: UserActivity
PK: user_id
SK: "ACTIVITY#type#timestamp"

Items:
{pk: user#123, sk: ACTIVITY#login#1695813000}
{pk: user#123, sk: ACTIVITY#post#1695813060}
{pk: user#123, sk: ACTIVITY#comment#1695813120}
{pk: user#123, sk: ACTIVITY#logout#1695813180}

Query(PK="user#123", SK begins_with "ACTIVITY#post")
→ All posts by user (without scanning all activity)
```

**Use case:** Activity tracking, filtering without scans.

---

## Sort Key Mistakes

### Mistake 1: Forgetting Sort Key Limitations
```
❌ WRONG:
Query(
  PK = "user#123",
  FK = "status",        // ERROR: Not a partition or sort key!
  value = "shipped"
)
→ Fails. Can't query by arbitrary attributes.

✓ CORRECT:
Query(
  PK = "user#123",
  SK begins_with "STATUS#shipped"
)
→ Works if SK contains status
```

### Mistake 2: Non-Sortable Sort Key
```
❌ WRONG:
SK: "ORDER#abc#ORDER#xyz#ORDER#123"

Lexicographic sort:
"ORDER#123"
"ORDER#abc"
"ORDER#xyz"

Not meaningful order (you wanted numeric).

✓ CORRECT:
SK: "ORDER#000001#ORDER#000010#ORDER#999999"

Or:
SK: 1  (just use number, format as string for display)
```

### Mistake 3: Overstuffing SK
```
❌ WRONG:
SK: "STATUS#pending#priority#high#created_at#1695813000#category#food#region#us-east#..."

Too much data → hard to query.
Can't do:
- Filter by category (have to begin_with "STATUS#pending#priority#high#created_at#...")
- Filter by region (have to begin_with "STATUS#pending#...")

✓ CORRECT:
Use GSI for alternate queries.
Keep SK simple: status, timestamp, or category.
```

---

## Sort Key Evolution

**Can you change sort key?** Yes, but hard.

**Workaround:** Use GSI (next lecture) with different SK.

---

## Query vs Scan

### Query (Fast)
```
Query(PK, SK condition)
→ O(log n) or O(n) within partition
→ Consistent performance
→ Charged per item read
```

### Scan (Slow)
```
Scan(filter condition)
→ O(n) entire table
→ Slow for large tables
→ Charged per item examined (even if filtered out)
```

**Rule:** Always Query. Never Scan if avoidable.

If you need to scan, you chose wrong sort key.

---

## Key Takeaway

Sort key enables:
1. Ordering items within partition
2. Range queries (>, <, between, begins_with)
3. Efficient filtering (without scans)

Design sort key to support your queries.

**Next:** [Access Patterns](./07-access-patterns.md)
