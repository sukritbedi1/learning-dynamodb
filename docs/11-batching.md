# Lecture 11: Batching & Pagination

## BatchGetItem

**Purpose:** Fetch up to 100 items in one request.

```go
response, _ := client.BatchGetItem(ctx, &dynamodb.BatchGetItemInput{
    RequestItems: map[string]types.KeysAndAttributes{
        "users": {
            Keys: []map[string]types.AttributeValue{
                {"user_id": &types.AttributeValueMemberS{Value: "1"}},
                {"user_id": &types.AttributeValueMemberS{Value: "2"}},
                {"user_id": &types.AttributeValueMemberS{Value: "3"}},
            },
        },
    },
})

// Result: items [1, 2, 3] in one request
// Cost: 3 RCU (instead of 3 separate GetItem = 3 RCU)
// Benefit: Network efficiency (one round-trip)
```

**Limits:**
- Up to 100 items per request
- Up to 16 MB total data
- Partial failures (one item fails, others succeed)

**Use case:** Fetch multiple items at once (e.g., load user + related orders).

---

## BatchWriteItem

**Purpose:** Write up to 25 items in one request.

```go
response, _ := client.BatchWriteItem(ctx, &dynamodb.BatchWriteItemInput{
    RequestItems: map[string][]types.WriteRequest{
        "orders": {
            {
                PutRequest: &types.PutRequest{
                    Item: map[string]types.AttributeValue{
                        "order_id": &types.AttributeValueMemberS{Value: "1"},
                        "total": &types.AttributeValueMemberN{Value: "99.99"},
                    },
                },
            },
            {
                PutRequest: &types.PutRequest{
                    Item: map[string]types.AttributeValue{
                        "order_id": &types.AttributeValueMemberS{Value: "2"},
                        "total": &types.AttributeValueMemberN{Value: "49.99"},
                    },
                },
            },
        },
    },
})

// Result: 2 items written in one request
// Cost: 2 WCU
// Benefit: One round-trip, faster
```

**Limits:**
- Up to 25 items per request
- Up to 16 MB total data
- Partial failures

**Use case:** Bulk inserts, imports, mass updates.

---

## Query Pagination

Queries return max 1 MB of data. Use `LastEvaluatedKey` to paginate.

```go
var allItems []map[string]types.AttributeValue
var lastKey map[string]types.AttributeValue

for {
    input := &dynamodb.QueryInput{
        TableName: "orders",
        KeyConditionExpression: "user_id = :uid",
        ExpressionAttributeValues: map[string]types.AttributeValue{
            ":uid": &types.AttributeValueMemberS{Value: "123"},
        },
        ExclusiveStartKey: lastKey,
        Limit: 100,  // Items per page
    }

    response, _ := client.Query(ctx, input)
    allItems = append(allItems, response.Items...)

    // Check if more pages exist
    if response.LastEvaluatedKey == nil {
        break  // No more pages
    }
    lastKey = response.LastEvaluatedKey
}
```

**Key concepts:**
- `Limit`: Max items per request (not total)
- `LastEvaluatedKey`: Resume from last item
- `ScannedCount`: Items examined (before filter)
- `Count`: Items returned (after filter)

---

## Scan Pagination

Same pagination mechanism as Query.

```go
response, _ := client.Scan(ctx, &dynamodb.ScanInput{
    TableName: "orders",
    ExclusiveStartKey: lastKey,
    Limit: 100,
})

// Process response.Items
// If response.LastEvaluatedKey != nil: more pages
```

**Warning:** Scan examines all items (expensive). Avoid for large tables.

---

## Pagination Patterns

### Pattern 1: Load All (Small Result Set)
```
No pagination needed if result < 1 MB.
Single Query/Scan call.
```

### Pattern 2: Web Pagination (Page 1, 2, 3)
```
Frontend: "Show page 1 (20 items)"
Backend: Query(Limit=20)
→ Return items + LastEvaluatedKey as pagination token
Frontend: Click "Next"
→ Send pagination token
Backend: Query(Limit=20, ExclusiveStartKey=token)
→ Return next 20 items + new token
```

### Pattern 3: Streaming (Process All)
```
// Process millions of items
for {
    Query(Limit=1000)
    Process items
    if LastEvaluatedKey == nil: break
}
```

---

## Performance Considerations

### Batch vs Single
```
100 GetItem calls: 100 network round-trips
1 BatchGetItem: 1 network round-trip

BatchGetItem is always faster (network bound).
```

### Pagination Overhead
```
Query returns 1000 items but you need 10.
Set Limit=10, stop after first page.
Saves data transfer.
```

### Query vs Scan
```
Query: Efficient (uses index)
Scan: Inefficient (examines all items)

For 1000 items:
Query(Limit=10): Examines ~10 items, returns 10 (cheap)
Scan(Limit=10, filter): Examines all 1000, returns 10 (expensive)
```

---

## Batch Failures

BatchGetItem/BatchWriteItem can partially fail:

```go
response, _ := client.BatchWriteItem(ctx, input)

// Check for failed items
if len(response.UnprocessedItems) > 0 {
    // Some items failed (throttled, error)
    // Retry later with exponential backoff
}
```

**Strategy:** Implement retry logic with exponential backoff.

---

## Key Takeaway

**Batch:** Multiple items in one request (network efficiency).
**Pagination:** Large result sets split into pages (memory efficiency).

Use both for optimal performance.

**Next:** [Transactions](./12-transactions.md)
