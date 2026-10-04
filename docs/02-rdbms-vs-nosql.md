# Lecture 2: RDBMS vs NoSQL — Mental Model Shift

## The Core Shift

You've built systems on **normalized relational schemas** (Postgres, MySQL). DynamoDB forces a complete mindset change.

### RDBMS Mindset (What You Know)

```
Start: Define tables & relationships
↓
Write: Normalize to avoid duplication
↓
Query: JOIN anything, anytime (flexible)
↓
Scale: Harder (joins are expensive at scale)
```

**Example: User with Orders**

```sql
-- RDS: Separate tables, JOIN when needed
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT REFERENCES users(id),
    total DECIMAL(10, 2),
    created_at TIMESTAMP
);

-- Query: JOIN to get user + orders
SELECT u.name, o.id, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.id = 123;
```

### DynamoDB Mindset (Access-Pattern-First)

```
Start: Define how you'll query the data
↓
Design: Schema around access patterns
↓
Write: Denormalize (duplicate data)
↓
Query: Fast; designed upfront
↓
Scale: Unlimited (partitioning is automatic)
```

**Example: Same User + Orders (DynamoDB)**

```
Access Pattern 1: "Get user by ID"
Access Pattern 2: "Get all orders for user"
Access Pattern 3: "Get order by order ID"

Then design table accordingly:

Table: UserOrders
{
  "PK": "USER#123",                    // Partition key
  "SK": "PROFILE",                     // Sort key
  "name": "Alice",
  "email": "alice@example.com"
}

{
  "PK": "USER#123",
  "SK": "ORDER#456",
  "total": 99.99,
  "created_at": 1695000000,
  "status": "shipped"
}

{
  "PK": "USER#123",
  "SK": "ORDER#789",
  "total": 49.99,
  "created_at": 1695001000,
  "status": "pending"
}

// Query Pattern 1: Get user profile
GetItem(PK="USER#123", SK="PROFILE")

// Query Pattern 2: Get all orders for user
Query(PK="USER#123", SK begins_with "ORDER#")

// Query Pattern 3: Get specific order
GetItem(PK="USER#123", SK="ORDER#456")
```

**Key Difference:** Data is denormalized and organized by access pattern, not by entity type.

---

## Three Critical Differences

### 1. Schema Flexibility

**RDS:**
```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- All rows must have phone (NULL or default)
-- Schema is centralized truth
```

**DynamoDB:**
```
No schema enforcement.

Item 1: {id, name, email}
Item 2: {id, name, email, phone}  -- Different attrs!
Item 3: {id, name, email, phone, address}

Application decides schema via code.
```

**Implication:** You can evolve schema without migrations. But: more application-side validation needed.

### 2. Query Model

**RDS:**
```sql
-- You can query any combination
SELECT * FROM orders WHERE user_id = 123 AND total > 100;
SELECT * FROM orders WHERE created_at > '2024-01-01';
SELECT * FROM orders WHERE status = 'shipped';

-- Index any column
CREATE INDEX idx_status ON orders(status);
```

**DynamoDB:**
```
You can only query by:
1. Partition Key (exact match)
2. Partition Key + Sort Key (exact/range)
3. Global Secondary Index (different PK/SK combo)

// Efficient:
Query(PK="USER#123")                    // ✓ Partition key
Query(PK="USER#123", SK > "ORDER#500")  // ✓ Range on sort key

// Inefficient/Impossible:
Scan(filter: total > 100)               // ✗ Scans entire table
Scan(filter: status = "shipped")        // ✗ Scans entire table
```

**Implication:** You design your table and indexes BEFORE you write queries.

### 3. Scaling

**RDS:**
```
Scaling Problem: JOIN on 10M users × 10M orders

SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- Takes minutes, expensive, can timeout
-- Solution: Add more CPU/RAM (vertical scaling, limited)
```

**DynamoDB:**
```
Same query: "Get user + all their orders"

Query(PK="USER#123", SK begins_with "ORDER#")
-- Returns in milliseconds, always
-- Handles 100M orders per user same speed
-- Solution: More partitions (automatic, horizontal)
```

**Implication:** DynamoDB scales linearly with partitions. Joins are your enemy.

---

## The Paradigm Shift

| RDBMS | DynamoDB |
|-------|----------|
| **Q:** What data do I have? **A:** Define tables | **Q:** How will I query it? **A:** Define access patterns |
| Query flexibility first | Performance first |
| Normalize (avoid duplication) | Denormalize (optimize for access) |
| Indexes are optional optimization | Indexes are part of core design |
| Scale by adding CPU/RAM | Scale by partition count (automatic) |
| Schema evolves after launch | Schema can evolve, but queries are fixed |

---

## Practical Consequence: Denormalization

You'll duplicate data. Example:

```
RDS (normalized):
users {id, name, email}
orders {id, user_id, total, status}

-- Get user + order status:
SELECT u.name, o.status FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.id = 456;

DynamoDB (denormalized):
Single item:
{
  PK: "ORDER#456",
  SK: "METADATA",
  user_id: "123",
  user_name: "Alice",        // DUPLICATE (denorm)
  user_email: "alice@...",   // DUPLICATE (denorm)
  total: 99.99,
  status: "shipped"
}

-- Get order + user name:
GetItem(PK="ORDER#456", SK="METADATA")
-- One call, instant
```

**Cost:** Data duplication (storage is cheap).
**Benefit:** Single, fast query (speed is valuable).

---

## Key Takeaway

**RDBMS:** "Give me all data, I'll query creatively."
**DynamoDB:** "Tell me your queries, I'll optimize the schema."

This is not a limitation. It's a feature. You trade flexibility for performance and scale.

**Next:** [Core Concepts](./03-core-concepts.md)
