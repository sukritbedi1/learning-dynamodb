# Lecture 3: Core Concepts

## Table

**Definition:** Collection of items. Like a file or relation in RDS.

```
DynamoDB Service
├── Table: users
│   ├── Item 1
│   ├── Item 2
│   └── Item 3
├── Table: orders
│   ├── Item 1
│   └── Item 2
└── Table: inventory
    └── Item 1
```

**Key differences from RDS table:**
- No schema enforced at DB level
- Each item can have different attributes
- Scaled independently (separate RCU/WCU)
- Can have multiple indexes (GSI, LSI)

---

## Item

**Definition:** Single record in a table. Like a row in RDS.

**Example item (JSON):**
```json
{
  "pk": "USER#123",
  "sk": "PROFILE",
  "name": "Alice",
  "email": "alice@example.com",
  "age": 30,
  "preferences": {
    "notifications": true,
    "theme": "dark"
  },
  "tags": ["vip", "early-adopter"],
  "created_at": 1695000000
}
```

**Key differences from RDS row:**
- Size limit: 400 KB per item
- Flexible attributes (no schema enforcement)
- Nested structures (maps, lists)
- Every item must have a Partition Key (PK)
- Sort Key optional but recommended

---

## Attribute

**Definition:** Single field within an item. Like a column in RDS.

**Types (DynamoDB supports):**

| Type | Symbol | Example | Notes |
|------|--------|---------|-------|
| String | S | `"Alice"` | Unicode, max 400 KB |
| Number | N | `30` or `99.99` | No size limit, precision to 38 digits |
| Binary | B | `<blob>` | Raw bytes, max 400 KB |
| Boolean | BOOL | `true` | Not a string; true/false only |
| Null | NULL | `null` | Represents missing/null |
| List | L | `[1, "two", true]` | Ordered, heterogeneous, max 400 KB |
| Map | M | `{a: 1, b: "two"}` | Nested object, max 400 KB |
| String Set | SS | `{"a", "b", "c"}` | Set of strings, unordered, unique |
| Number Set | NS | `{1, 2, 3}` | Set of numbers, unordered, unique |
| Binary Set | BS | `{blob1, blob2}` | Set of binaries, unordered, unique |

**Example item with various types:**
```json
{
  "pk": "USER#123",
  "sk": "PROFILE",
  "name": "Alice",                    // String (S)
  "age": 30,                          // Number (N)
  "is_active": true,                  // Boolean (BOOL)
  "avatar": "<binary>",               // Binary (B)
  "tags": ["vip", "customer"],        // List (L)
  "preferences": {                    // Map (M)
    "theme": "dark",
    "notifications": true
  },
  "hobbies": {"reading", "coding"},   // String Set (SS)
  "favorite_numbers": {1, 7, 42},     // Number Set (NS)
  "last_login": null                  // Null (NULL)
}
```

**Important:**
- Sets must contain same type (all strings, all numbers, or all binaries)
- Empty sets not allowed
- No nested sets (can't have set of sets)
- Order in sets not guaranteed

---

## Partition Key (Primary Key - Part 1)

**Definition:** Attribute that determines which partition stores the item.

Also called: **Hash Key** or **Distribution Key**

**Purpose:** DynamoDB uses partition key to distribute data across partitions.

```
Table: users
├── Partition 1 (hash(pk) % 256 = 0)
│   ├── Item: pk=USER#001
│   └── Item: pk=USER#257
├── Partition 2 (hash(pk) % 256 = 1)
│   ├── Item: pk=USER#002
│   └── Item: pk=USER#258
└── Partition 3 (hash(pk) % 256 = 2)
    ├── Item: pk=USER#003
    └── Item: pk=USER#259
```

**How it works:**
1. You provide partition key value
2. DynamoDB hashes it
3. Hash determines which partition stores item
4. Different keys → different partitions → parallelizable

**Example:**
```
GetItem(Table="users", PK="USER#123")
→ hash("USER#123") → partition 42
→ Fetch from partition 42 only (fast, O(1))

GetItem(Table="users", PK="USER#456")
→ hash("USER#456") → partition 7
→ Fetch from partition 7 only (fast, O(1))
```

**Constraints:**
- Every table must have a partition key
- Partition key must be String, Number, or Binary
- Partition key is immutable (can't change after insert)
- Cannot be null

---

## Sort Key (Primary Key - Part 2)

**Definition:** Optional second part of primary key. Enables range queries.

Also called: **Range Key** or **Clustering Key**

**Purpose:** Orders items within a partition.

```
Table: users (PK = "USER#123", SK = order time)
Partition for USER#123:
├── SK: 1695000000 → Order shipped
├── SK: 1695001000 → Order pending
└── SK: 1695002000 → Order cancelled

Query(PK="USER#123")
→ Returns all items in this partition, sorted by SK
```

**Query capabilities with Sort Key:**
```
Query(PK="USER#123", SK = "ORDER#001")
→ Exact match (one item)

Query(PK="USER#123", SK begins_with "ORDER#")
→ All orders for user

Query(PK="USER#123", SK between "ORDER#100" and "ORDER#200")
→ Orders in range

Query(PK="USER#123", SK > "ORDER#500")
→ Orders after ORDER#500

Query(PK="USER#123")
→ All items in partition, sorted by SK
```

**Constraints:**
- Sort key is optional (table can have only PK)
- Sort key must be String, Number, or Binary
- Sort key is immutable
- Cannot be null (if defined)
- Items with same PK are sorted by SK

---

## Primary Key Combinations

### Only Partition Key
```
Table: User Sessions
PK: session_id

GetItem(PK="sess_xyz789")
→ One session, fast

Query(PK="sess_xyz789")
→ Invalid (Query needs range)
```

### Partition Key + Sort Key (Most Common)
```
Table: Orders
PK: user_id
SK: order_id

GetItem(PK="user#123", SK="order#456")
→ Exact order

Query(PK="user#123", SK begins_with "order#")
→ All orders for user

Query(PK="user#123", SK > "order#500")
→ Orders after 500
```

---

## Key Design Patterns

### Pattern 1: Entity as PK, Type as SK
```
Table: UserData
PK: "USER#123"
SK: "PROFILE"        → User metadata
SK: "SETTINGS"       → User preferences
SK: "ORDER#456"      → Embedded order

Query(PK="USER#123")
→ Returns profile + settings + all orders for user
```

**Use case:** All data for one user in one partition (one query).

### Pattern 2: Timestamp as SK (Time Series)
```
Table: Metrics
PK: "METRIC#cpu_usage"
SK: 1695000000       → Value at time 1695000000
SK: 1695001000       → Value at time 1695001000
SK: 1695002000       → Value at time 1695002000

Query(PK="METRIC#cpu_usage", SK > 1695000000)
→ All metrics after timestamp (monitoring, time-series)
```

**Use case:** Metrics, logs, events over time.

### Pattern 3: Composite SK (Hierarchy)
```
Table: Store Inventory
PK: "STORE#NYC"
SK: "PRODUCT#MILK#2L"    → 2L milk
SK: "PRODUCT#MILK#1L"    → 1L milk
SK: "PRODUCT#BREAD#WW"   → Whole wheat bread

Query(PK="STORE#NYC", SK begins_with "PRODUCT#MILK")
→ All milk products in NYC store
```

**Use case:** Inventory, categories, hierarchies.

---

## Hot Key Problem

**Definition:** Single partition key that gets queried too often.

**Example (BAD):**
```
Table: Inventory
PK: "GLOBAL_INVENTORY"  // ALL products in one partition
SK: product_id

Every request:
Query(PK="GLOBAL_INVENTORY", SK="product#123")
Query(PK="GLOBAL_INVENTORY", SK="product#456")
Query(PK="GLOBAL_INVENTORY", SK="product#789")

Problem:
All reads → one partition → throttled
Other partitions idle
```

**Solution:** Distribute by region/store/category
```
Table: Inventory
PK: "STORE#NYC"          // One partition per store
SK: product_id

Different stores:
Query(PK="STORE#NYC", SK="product#123")
Query(PK="STORE#LA", SK="product#123")
Query(PK="STORE#CHI", SK="product#123")

Better:
Reads distributed across partitions
Each partition handles its store's traffic
```

**Lesson:** Choose partition key to distribute queries evenly.

---

## Summary Table

| Concept | Purpose | Example |
|---------|---------|---------|
| **Table** | Collection of items | `users`, `orders` |
| **Item** | Single record | `{pk: USER#123, sk: PROFILE, ...}` |
| **Attribute** | Single field | `name`, `age`, `tags` |
| **Partition Key** | Determines partition | `user_id` |
| **Sort Key** | Orders within partition | `created_at` |
| **Primary Key** | PK + SK combo (unique per table) | `(user_id, order_id)` |

---

## Key Takeaway

DynamoDB structure is simple: Tables → Items → Attributes → Keys.

Power comes from key design:
- **Partition Key:** Determines which partition (distribution)
- **Sort Key:** Enables range queries (ordering)
- Together: They define your query patterns

Choose them wrong → slow queries.
Choose them right → instant queries at any scale.

**Next:** [Data Types & Schema](./04-data-types.md)
