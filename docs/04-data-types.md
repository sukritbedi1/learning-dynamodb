# Lecture 4: Data Types & Schema Design

## DynamoDB Data Types (Deep Dive)

### Scalar Types

#### 1. String (S)
```
"name": "Alice"
"email": "alice@example.com"
"description": "Very long text..."

Size: Up to 400 KB
Encoding: UTF-8
Comparison: Lexicographic (alphabetical order)
```

**Use case:** Names, emails, descriptions, IDs.

#### 2. Number (N)
```
"age": 30
"price": 99.99
"score": -5
"large_number": 999999999999999999999999999999999999

Precision: Up to 38 digits
Range: ±9.9999999999999999999999999999999999E+125
No Size Limit

Comparison: Numeric (not string)
```

**Important:** Numbers are stored as decimal (not float). 99.99 ≠ "99.99".

**Use cases:** Age, price, counts, timestamps.

#### 3. Binary (B)
```
Raw bytes, e.g., encrypted data, images (base64 encoded).

"avatar": "<base64-encoded-image>"
"encrypted_key": "<binary-blob>"

Size: Up to 400 KB
Comparison: Byte-by-byte (lexicographic on bytes)
```

**Use case:** Blobs, encrypted data, images, files.

#### 4. Boolean (BOOL)
```
"is_active": true
"is_verified": false

Only two values: true or false
Not a string ("true" ≠ true)
```

**Use case:** Flags, switches, status booleans.

#### 5. Null (NULL)
```
"middle_name": null

Represents absence of value.
Different from empty string ("") or 0.
```

**Use case:** Optional fields that are unset.

---

### Collection Types

#### 6. List (L)
```
Ordered collection. Can mix types.

"tags": ["vip", "customer", "premium"]
"scores": [100, 95, 87]
"mixed": [1, "two", true, {nested: "map"}]

Size: Each item max 400 KB; list max 400 KB total
Order: Preserved
Indexing: By position (0, 1, 2, ...)
Comparison: Element-by-element

Access in code:
list[0] → "vip"
list[2] → "premium"
```

**Use case:** Tags, arrays, nested data.

#### 7. Map (M)
```
Unordered nested object.

"address": {
  "street": "123 Main St",
  "city": "NYC",
  "zip": "10001"
}

"settings": {
  "theme": "dark",
  "notifications": true,
  "language": "en"
}

Size: Max 400 KB
Nesting: Can nest maps within maps
Access: By key

address.city → "NYC"
settings.notifications → true
```

**Use case:** Nested objects, structured data, settings.

---

### Set Types

#### 8. String Set (SS)
```
Unordered collection of unique strings.

"tags": {"vip", "customer", "premium"}
"hobbies": {"reading", "coding", "gaming"}

Uniqueness: Duplicates removed automatically
Order: Not guaranteed (set semantics)
Size: No size limits on set; items must be strings

Important Constraints:
- Cannot contain duplicate values
- Cannot be empty
- Cannot nest sets
```

**Use case:** Tags, categories, user roles.

#### 9. Number Set (NS)
```
Unordered collection of unique numbers.

"favorite_numbers": {1, 7, 42, 100}
"ratings": {4, 5, 3, 5}  → {4, 5, 3} (duplicates removed)

Same constraints as SS.
```

**Use case:** IDs, scores, ratings.

#### 10. Binary Set (BS)
```
Unordered collection of unique binary values.

"encrypted_keys": {blob1, blob2, blob3}

Same constraints as SS.
```

**Use case:** Encrypted tokens, signatures.

---

## Type Conversion & Common Mistakes

### Mistake 1: String vs Number
```
❌ WRONG:
"age": "30"        // String
"price": "99.99"   // String

Comparisons fail:
Query(SK > "30")   // Returns "4", "5", "50", "99" (lexicographic!)

✓ CORRECT:
"age": 30          // Number
"price": 99.99     // Number

Query(SK > 30)     // Returns 31, 40, 100 (numeric!)
```

### Mistake 2: Empty Sets
```
❌ WRONG:
"tags": {}         // Empty set (NOT allowed)

✓ CORRECT:
"tags": null       // Use null instead
// Or omit attribute entirely
```

### Mistake 3: Timestamp as String
```
❌ WRONG:
"created_at": "2024-09-27T10:30:00Z"

Can't do range queries:
Query(created_at > "2024-09-01")  // Lexicographic; wrong results

✓ CORRECT:
"created_at": 1695813000          // Unix timestamp (Number)

Query(created_at > 1695800000)    // Numeric comparison
```

---

## Schema Design Principles

### Principle 1: No Schema Enforcement (Flexibility)

RDS enforces schema at DB level:
```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(100) NOT NULL
);

INSERT INTO users (name, email) VALUES ('Alice', 'alice@...');
-- OK

INSERT INTO users (name) VALUES ('Bob');
-- ERROR: email required
```

DynamoDB doesn't:
```
Item 1: {pk, sk, name, email}
Item 2: {pk, sk, name}           // Missing email
Item 3: {pk, sk, name, email, phone}  // Extra phone

All valid. No error.
Application must validate.
```

**Consequence:** Be explicit in code about required fields.

### Principle 2: Denormalization for Access Patterns

RDS: Normalize first
```sql
CREATE TABLE users (id, name, email);
CREATE TABLE orders (id, user_id, total, status);

Query:
SELECT u.name, o.id, o.status
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.id = 123;
```

DynamoDB: Denormalize for each access pattern
```
Access Pattern 1: Get user profile
{
  pk: "USER#123",
  sk: "PROFILE",
  name: "Alice",
  email: "alice@..."
}

Access Pattern 2: Get user + all orders
{
  pk: "USER#123",
  sk: "ORDER#456",
  name: "Alice",              // DUPLICATE (denormalized)
  email: "alice@...",         // DUPLICATE (denormalized)
  total: 99.99,
  status: "shipped"
}

Access Pattern 3: Get order by order ID
{
  pk: "ORDER#456",
  sk: "METADATA",
  user_name: "Alice",         // DUPLICATE (denormalized)
  user_email: "alice@...",    // DUPLICATE (denormalized)
  total: 99.99,
  status: "shipped"
}
```

**Trade-off:** Storage grows, but queries are fast and simple.

### Principle 3: Design for Queries, Not Data

RDS: Design for data relationships
```
Ask: What entities do I have? (users, orders, products)
Then: How will I query them?
```

DynamoDB: Design for queries
```
Ask: How will I query the data?
- Get user by ID?
- Get all orders for user?
- Get order by order ID?
- Get inventory for store?

Then: Design schema to support these queries
```

**Consequence:** Same data, different schemas depending on queries.

---

## Schema Design Patterns

### Pattern 1: Single Table Design
```
ONE table for ALL entities (modern approach)

Table: AppData
{
  pk: "USER#123",
  sk: "PROFILE",
  // ... user fields
}

{
  pk: "USER#123",
  sk: "ORDER#456",
  // ... order fields
}

{
  pk: "ORDER#456",
  sk: "METADATA",
  // ... order metadata
}

{
  pk: "PRODUCT#789",
  sk: "INVENTORY",
  // ... inventory fields
}

Advantage: Single round-trip for related data
Disadvantage: Complex schema, careful key design required
```

### Pattern 2: Multiple Tables (Traditional Approach)
```
Separate table per entity

Table: Users
{pk: user_id, email, name}

Table: Orders
{pk: order_id, user_id, total, status}

Table: Products
{pk: product_id, name, price}

Advantage: Simple, clear schema
Disadvantage: Need app-side joins, more queries
```

### Pattern 3: Hybrid (Best of Both)
```
Core entities in separate tables:
Table: Users (user_id, name, email)
Table: Orders (order_id, user_id, total)

Performance-critical data denormalized:
Table: UserOrders (pk: user_id, sk: order_id, with denormalized user data)
→ GSI for alternate queries

Advantage: Simple + performant
```

---

## Schema Evolution

### Adding New Attributes
```
✓ Easy: Just write new attributes on new items

Old item:
{pk, sk, name, email}

New item:
{pk, sk, name, email, phone, address}

No migration needed.
```

### Renaming Attributes
```
⚠️ Hard: Requires scanning and rewriting items

Option 1: Leave old, add new
{pk, sk, email, email_new}
→ Migrate app to use email_new
→ Clean up old email later

Option 2: Scan and rewrite (expensive for large tables)
```

### Removing Attributes
```
✓ Easy: Stop writing them, leave old ones

Items still have the field, but you ignore it.
Eventually they fade as items are updated.
```

### Changing Type
```
⚠️ Hard: Can't change "age": "30" (string) to "age": 30 (number)

Solution: Use different attribute name
"age": "30"
"age_numeric": 30

Migrate, then remove old attribute.
```

---

## Schema Design Checklist

Before launching:
- [ ] Define all access patterns (queries you'll run)
- [ ] Choose partition key (even distribution?)
- [ ] Choose sort key (enables range queries?)
- [ ] Plan denormalization (which data duplicates where?)
- [ ] Plan GSI/LSI (alternate query patterns?)
- [ ] Estimate size per item × expected items
- [ ] Estimate queries per second (RCU/WCU planning)
- [ ] Plan evolution (adding fields later?)

---

## Key Takeaway

DynamoDB is flexible in structure but rigid in queries.

Data types are straightforward (7 scalars + 3 collections). The hard part is schema design: choosing keys and denormalization to enable your access patterns.

**Next:** [Partition Keys & Distribution](./05-partition-keys.md)
