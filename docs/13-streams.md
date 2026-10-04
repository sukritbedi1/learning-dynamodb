# Lecture 13: Streams & Change Data Capture

## DynamoDB Streams

**Definition:** Ordered log of item modifications (create, update, delete).

Enables: Trigger Lambda, CDC, real-time processing.

```
Item updated → Stream record created → Lambda triggered → Process change
```

---

## Stream Record

```
{
  eventID: "1",
  eventVersion: "1.0",
  dynamodb: {
    Keys: {user_id: {S: "123"}},
    NewImage: {
      user_id: {S: "123"},
      name: {S: "Alice"},
      email: {S: "alice@example.com"}
    },
    OldImage: {
      user_id: {S: "123"},
      name: {S: "Alice"},
      email: {S: "old@example.com"}
    },
    StreamViewType: "NEW_AND_OLD_IMAGES"
  },
  eventName: "MODIFY",  // INSERT, MODIFY, DELETE
  eventSource: "aws:dynamodb",
  awsRegion: "us-east-1"
}
```

---

## Stream View Types

| View | Contains | Use Case |
|------|----------|----------|
| KEYS_ONLY | Keys only | Track changes (not values) |
| NEW_IMAGE | New item | What changed (for updates) |
| OLD_IMAGE | Old item | Previous state (audit) |
| NEW_AND_OLD_IMAGES | Both | Full before/after (CDC, sync) |

---

## Enable Streams

```go
// Create table with streams enabled
streamSpec := &types.StreamSpecification{
    StreamEnabled: true,
    StreamViewType: types.StreamViewTypeNewAndOldImages,
}

CreateTableInput.StreamSpecification = streamSpec
```

---

## Lambda Trigger

```go
// Configure Lambda to process stream

EventSourceMappings: {
    EventSource: "dynamodb",
    Arn: "arn:aws:dynamodb:...:table/orders/stream/...",
    StartingPosition: "TRIM_HORIZON",  // Start from oldest
    BatchSize: 100,  // Items per Lambda invocation
    FunctionName: "ProcessOrderChanges",
}
```

Lambda receives batch of stream records:
```go
func ProcessOrderChanges(ctx context.Context, event events.DynamoDBEvent) {
    for _, record := range event.Records {
        switch record.EventName {
        case "INSERT":
            // New order created
        case "MODIFY":
            // Order updated
        case "DELETE":
            // Order deleted
        }
    }
}
```

---

## Use Cases

### Case 1: Real-Time Analytics
```
Order created → Stream → Lambda → Write to Analytics table
User profile updated → Stream → Lambda → Update cache

Real-time insights without polling.
```

### Case 2: CDC (Change Data Capture)
```
DynamoDB → Stream → Lambda → Kafka → Data warehouse

Sync DynamoDB changes to external systems.
```

### Case 3: Audit Trail
```
Any change → Stream → Lambda → Store in audit table

Who changed what, when.
```

### Case 4: Notification
```
Order status changed → Stream → Lambda → Send email/SMS

Automatic notification on changes.
```

---

## Stream Ordering

```
Records from same partition: Ordered
Records from different partitions: Not guaranteed ordered

Example:
Item 1 (partition A) modified
Item 2 (partition A) modified
→ Strict order

Item 1 (partition A) modified
Item 3 (partition B) modified
→ No guaranteed order (but usually fast)
```

---

## Limitations

- 24-hour retention (records older than 24h deleted)
- Update Lambda or Kinesis as consumers
- No SQL queries on streams

---

## Key Takeaway

Streams enable: Real-time CDC, notifications, analytics.

Enable for tables requiring change tracking. Process with Lambda or Kinesis.

**Next:** [Global Tables & Replication](./14-global-tables.md)
