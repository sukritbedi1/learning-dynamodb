# Lecture 12: Transactions

## ACID Guarantees

DynamoDB transactions provide ACID on up to 25 items (limited).

| Property | Guarantee |
|----------|-----------|
| **Atomicity** | All succeed or all fail (no partial) |
| **Consistency** | Data valid after transaction |
| **Isolation** | Concurrent transactions don't interfere |
| **Durability** | Committed data survives failures |

---

## TransactWriteItems

Write multiple items atomically.

```go
response, _ := client.TransactWriteItems(ctx, &dynamodb.TransactWriteItemsInput{
    TransactItems: []types.TransactWriteItem{
        {
            Put: &types.Put{
                TableName: "orders",
                Item: map[string]types.AttributeValue{
                    "order_id": &types.AttributeValueMemberS{Value: "123"},
                    "total": &types.AttributeValueMemberN{Value: "99.99"},
                },
            },
        },
        {
            Update: &types.Update{
                TableName: "inventory",
                Key: map[string]types.AttributeValue{
                    "product_id": &types.AttributeValueMemberS{Value: "milk"},
                },
                UpdateExpression: "SET qty = qty - :qty",
                ExpressionAttributeValues: map[string]types.AttributeValue{
                    ":qty": &types.AttributeValueMemberN{Value: "1"},
                },
            },
        },
    },
})

// If both succeed: committed
// If either fails: both rolled back
```

**Operations allowed:**
- Put
- Update
- Delete
- ConditionCheck (conditional)

**Limits:**
- Up to 25 items
- Up to 4 MB total
- Cannot span multiple items of same key (all different items)

---

## TransactGetItems

Read multiple items atomically (consistent snapshot).

```go
response, _ := client.TransactGetItems(ctx, &dynamodb.TransactGetItemsInput{
    TransactItems: []types.TransactGetItem{
        {
            Get: &types.Get{
                TableName: "orders",
                Key: map[string]types.AttributeValue{
                    "order_id": &types.AttributeValueMemberS{Value: "123"},
                },
            },
        },
        {
            Get: &types.Get{
                TableName: "users",
                Key: map[string]types.AttributeValue{
                    "user_id": &types.AttributeValueMemberS{Value: "456"},
                },
            },
        },
    },
})

// Returns [order, user] in consistent snapshot
```

---

## Conditional Writes

Transactions support conditions.

```go
{
    Update: &types.Update{
        TableName: "inventory",
        Key: map[string]types.AttributeValue{
            "product_id": &types.AttributeValueMemberS{Value: "milk"},
        },
        UpdateExpression: "SET qty = qty - :qty",
        ConditionExpression: "qty > :zero",  // Fail if qty <= 0
        ExpressionAttributeValues: map[string]types.AttributeValue{
            ":qty": &types.AttributeValueMemberN{Value: "1"},
            ":zero": &types.AttributeValueMemberN{Value: "0"},
        },
    },
}

// If qty <= 0: entire transaction fails
```

---

## Use Cases

### Case 1: Money Transfer
```
Transfer $50 from Alice to Bob

TransactWriteItems:
1. Deduct $50 from Alice's balance (condition: balance >= 50)
2. Add $50 to Bob's balance

Both succeed or both fail. No partial transfer.
```

### Case 2: Inventory Decrement
```
User buys milk, stock decreases, order created.

TransactWriteItems:
1. Create order (condition: order doesn't exist)
2. Decrement inventory (condition: qty > 0)
3. Update user's order count

All succeed atomically.
```

### Case 3: Multi-Step Process
```
Checkout:
1. Validate coupon (ConditionCheck)
2. Create order
3. Decrement inventory
4. Update user's loyalty points

All atomic or all fail.
```

---

## Limitations

**Can't use TransactWriteItems for:**
- Cross-partition transactions (different partition keys)
- 25+ items
- Queries or scans

**Workaround:** App-side orchestration (manual rollback).

---

## Error Handling

```go
response, err := client.TransactWriteItems(ctx, input)

if err != nil {
    // Transaction failed
    // Check specific error type
    // Could be ValidationException, TransactionConflictException, etc.
    // Retry or handle manually
}

// Success: all items written
```

---

## Key Takeaway

Transactions enable multi-item ACID guarantees for up to 25 items.

Use for:
- Atomic multi-step operations
- Conditional writes
- Consistent reads

Avoid for:
- Large transactions (25+ items)
- Complex application logic (use app-side coordination)

**Next:** [Streams & Change Data Capture](./13-streams.md)
