# Lecture 9: Read/Write Consistency

## Strong vs Eventual Consistency

### Strong Consistency
```
Write data.
Read immediately.
→ You always see latest value.

Cost: More expensive (extra replication checks)
Latency: Slightly higher
```

### Eventual Consistency
```
Write data (replicated in background).
Read immediately.
→ Might see old value (briefly).

After a few milliseconds:
→ All replicas updated
→ Future reads see latest

Cost: Cheaper (simpler replication)
Latency: Lower
```

---

## How DynamoDB Replicates

```
Item written to primary partition:
{user_id: 123, name: "Alice"}

Immediately:
- Primary partition has latest
- Replica partitions (for durability) behind

Read from primary → see latest (strong)
Read from replica → might see old (eventual)
```

---

## Consistency Options

### GetItem
```go
// Strong Consistency
response, _ := client.GetItem(ctx, &dynamodb.GetItemInput{
    TableName: "users",
    Key: map[string]types.AttributeValue{
        "user_id": &types.AttributeValueMemberS{Value: "123"},
    },
    ConsistentRead: true,  // ← Strong
})

// Eventual Consistency (default)
response, _ := client.GetItem(ctx, &dynamodb.GetItemInput{
    TableName: "users",
    Key: map[string]types.AttributeValue{
        "user_id": &types.AttributeValueMemberS{Value: "123"},
    },
    // ConsistentRead: false (default)
})
```

### Query
```go
// Strong Consistency
response, _ := client.Query(ctx, &dynamodb.QueryInput{
    TableName: "users",
    KeyConditionExpression: "user_id = :uid",
    ExpressionAttributeValues: map[string]types.AttributeValue{
        ":uid": &types.AttributeValueMemberS{Value: "123"},
    },
    ConsistentRead: true,  // ← Strong
})

// Eventual Consistency (default)
```

### Scan
```
Scan doesn't support strong consistency (by design).
Always eventual.
```

---

## When to Use Each

### Strong Consistency
```
Use when: Data must be latest
- Financial transactions (show accurate balance)
- Inventory (show current stock)
- Password verification (check latest password)

Cost: 2x RCU (two reads instead of one)
```

### Eventual Consistency
```
Use when: Slight staleness acceptable (milliseconds)
- User profile (name, email) - unlikely to change mid-action
- Game leaderboard (players won't mind 1-second-old rankings)
- Analytics (aggregated data, staleness OK)
- Personalization (caches anyway)

Cost: 1x RCU (cheaper)
```

**Rule:** Default to eventual. Use strong only when necessary.

---

## Real-World Example: High-Throughput Platform

```
Inventory Check:
- User selects item
- App checks: Is item in stock?
- Answer must be current (not seconds old)
→ Strong consistency required
→ 2x RCU cost

User Profile:
- Load user name, preferences
- Slight staleness OK (unlikely to change mid-session)
→ Eventual consistency OK
→ 1x RCU cost
```

---

## Consistency Guarantees

### Durability
```
Write succeeds → data is durable.
Replicated immediately to 3 locations.
Even if one fails, data survives.
```

### Atomicity
```
Single item write → all-or-nothing.
Partial writes impossible.
```

### Item-Level Consistency
```
Single item:
- Strong: Always latest
- Eventual: Eventually latest

Multiple items:
- No atomicity across items (unless TransactWriteItems)
```

---

## Consistency Doesn't Mean Ordering

```
❌ Wrong assumption:
Write 1, Write 2, then Read
→ Reads both latest

✓ Correct:
Write 1 (succeeds)
Write 2 (succeeds)
Read with Eventual Consistency
→ Might see Write 1 only (Write 2 not yet replicated)

Multiple updates need strong consistency or transactions.
```

---

## Key Takeaway

**Strong Consistency:** Always latest, costs 2x RCU.
**Eventual Consistency:** Shortly latest, costs 1x RCU (default).

Choose based on staleness tolerance, not on fear.
Default to eventual for cost. Use strong only when necessary.

**Next:** [Billing & Cost Optimization](./10-billing.md)
