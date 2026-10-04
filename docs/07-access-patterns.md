# Lecture 7: Access Patterns

## What are Access Patterns?

**Definition:** Specific queries your application will run.

Instead of: "I have users, orders, products"
Think: "I need to get user by ID", "I need to get all orders for a user"

Access patterns drive everything: table design, keys, indexes.

---

## Why Access Patterns First?

### RDBMS Approach (Entity-First)
```
1. Define entities (users, orders, products)
2. Normalize relationships (foreign keys)
3. Deploy
4. Write queries (hopes they're fast)
5. If slow: add indexes, restructure tables (expensive)
```

### DynamoDB Approach (Query-First)
```
1. Define access patterns (how will we query?)
2. Design schema to support patterns (keys, denormalization)
3. Deploy
4. Queries are fast (by design)
5. No retrospective optimization needed
```

---

## Identifying Access Patterns

Ask:
- "How will we retrieve data?"
- "Which attributes will we filter on?"
- "Do we need ranges (after this time, between X and Y)?"
- "What's the primary way to access this data?"
- "Are there alternate ways (secondary access patterns)?"

---

## Access Pattern Examples

### E-Commerce (Real-Time Delivery)

**Primary Access Patterns:**

1. **Get user by ID**
```
Query: GetItem(user_id)
Used: Login, profile page, every user action
PK: user_id
```

2. **Get all orders for a user**
```
Query: Query(user_id, order_id)
Used: Order history page
PK: user_id
SK: order_id or created_at
```

3. **Get specific order**
```
Query: Query(user_id, order_id) or GetItem(order_id, metadata)
Used: Order detail page
Need: Fast exact lookup by order ID
```

4. **Get orders by status**
```
Query: Query(user_id, begins_with "status#pending")
Used: My orders (pending vs delivered)
PK: user_id
SK: "status#pending#created_at#..." or use GSI
```

5. **Get inventory for store**
```
Query: Query(store_id, product_id)
Used: Store page (show available products)
PK: store_id
SK: product_id
```

6. **Search products by category**
```
Query: Query(category, product_name) or Scan(filter: category)
Used: Browse page
GSI: category as PK, product_name as SK
```

---

## Schema Design from Access Patterns

**Access Patterns (defined above):**
1. Get user by ID
2. Get all orders for user
3. Get specific order
4. Get orders by status
5. Get inventory by store
6. Search by category

**Table Design:**

```
Table: UserData (Primary)
PK: user_id
SK: type (PROFILE, ORDER#..., SETTINGS)

Items:
{pk: user#123, sk: PROFILE, name, email, ...}
{pk: user#123, sk: ORDER#456, order_data, status}
{pk: user#123, sk: ORDER#789, order_data, status}
{pk: user#123, sk: SETTINGS, preferences}

Access Pattern 1 (Get user):
GetItem(user#123, PROFILE) ✓

Access Pattern 2 (Get user orders):
Query(user#123, begins_with ORDER#) ✓

Access Pattern 3 (Get specific order):
GetItem(user#123, ORDER#456) ✓

Access Pattern 4 (Get pending orders):
Query(user#123, begins_with "ORDER#") + filter by status ✓
(Or GSI if status filtering is critical)
```

```
Table: StoreInventory
PK: store_id
SK: product_id

Items:
{pk: store#NYC, sk: product#milk, qty, price}
{pk: store#NYC, sk: product#bread, qty, price}
{pk: store#LA, sk: product#milk, qty, price}

Access Pattern 5 (Get inventory):
Query(store#NYC) ✓
```

```
GSI: ProductCategory
PK: category
SK: product_id

Items:
{pk: dairy, sk: product#milk, ...}
{pk: dairy, sk: product#cheese, ...}
{pk: bakery, sk: product#bread, ...}

Access Pattern 6 (Search by category):
Query(dairy) ✓
```

---

## Access Pattern Documentation

Template:
```
Access Pattern: Get user by ID
- Query: GetItem(user_id)
- Frequency: Very high (every user action)
- Latency SLA: <10ms
- Implemented by: PK = user_id
- Alternative: None
```

```
Access Pattern: Get all orders for user
- Query: Query(user_id, begins_with ORDER#)
- Frequency: High (user clicks "My Orders")
- Latency SLA: <100ms
- Implemented by: PK = user_id, SK = ORDER#...
- Alternative: Could filter by status here (Scan is too slow)
```

```
Access Pattern: Search by category
- Query: Query(category, product_id)
- Frequency: High (browse page)
- Latency SLA: <200ms
- Implemented by: GSI with PK = category, SK = product_id
- Alternative: Scan whole table (too slow, not acceptable)
```

---

## Common Access Pattern Mistakes

### Mistake 1: Too Many Patterns
```
❌ WRONG: Trying to support every possible query

Access Patterns:
- Get user by ID
- Get user by email
- Get user by phone
- Get orders by status
- Get orders by total price
- Get orders by date
- ...10 more patterns...

Problem: Table design becomes complex, compromises.
```

**Solution:** Support 3-5 primary patterns. Use Scan or Athena for ad-hoc queries.

### Mistake 2: Forgetting Query Frequency
```
❌ WRONG: Designing for rare queries

Access Pattern: Get all users in a city
Frequency: Daily report (once)
Implemented by: GSI on city

Problem: GSI has storage cost, scaling cost, for a once-daily query.
```

**Solution:** Use Scan or Athena for rare queries. Reserve indexes for high-frequency patterns.

### Mistake 3: Not Documenting Patterns
```
❌ WRONG: Design schema without documenting WHY

Months later: "Why do we have this GSI? Can we remove it?"
"Not sure. Might break something."
```

**Solution:** Document each pattern: query, frequency, why it exists.

---

## Real-World: High-Throughput Commerce Access Patterns

(Example: E-commerce platform with millions of daily transactions)

```
1. Get user profile
   - GetItem(user_id, PROFILE)
   - Frequency: Every login, every page load
   - Critical: Must be <50ms

2. Get user's active orders
   - Query(user_id, begins_with "ORDER#ACTIVE")
   - Frequency: High (order page, home screen)
   - Critical: Must show within 100ms

3. Get order details
   - GetItem(order_id, METADATA)
   - Frequency: High (user clicks order)
   - Critical: Must be <50ms

4. Get available inventory for location
   - Query(location_id, begins_with "PRODUCT#")
   - Frequency: Very high (every product view)
   - Critical: Must be <30ms (user sees products loading)

5. Search products by category
   - Query(category, product_id)
   - Frequency: Medium (user browses)
   - Critical: Must be <500ms (page load)
   - Implemented by: GSI

6. Get fulfillment options
   - Query(address_id, begins_with "OPTION#")
   - Frequency: High (checkout)
   - Critical: Must be <100ms
```

---

## Key Takeaway

**Design Schema for Access Patterns, not Entities**

1. List your queries
2. Identify frequency and latency needs
3. Design PK/SK/GSI to support them
4. Deploy
5. Queries are fast by design

Identify access patterns FIRST. Everything else follows.

**Next:** [Secondary Indexes (GSI, LSI)](./08-secondary-indexes.md)
