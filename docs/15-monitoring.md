# Lecture 15: Monitoring & Observability

## CloudWatch Metrics

DynamoDB publishes metrics automatically.

| Metric | Unit | Meaning |
|--------|------|---------|
| ConsumedReadCapacityUnits | Count | Actual RCU used |
| ConsumedWriteCapacityUnits | Count | Actual WCU used |
| ProvisionedReadCapacityUnits | Count | RCU provisioned |
| ProvisionedWriteCapacityUnits | Count | WCU provisioned |
| UserErrors | Count | Application errors (bad requests) |
| SystemErrors | Count | AWS errors |
| SuccessfulRequestLatency | Milliseconds | Request time |
| Throttling | Count | Requests rejected (capacity exceeded) |

---

## Key Metrics to Monitor

### Read Capacity
```
ConsumedReadCapacityUnits vs ProvisionedReadCapacityUnits
→ If consumed > provisioned: Throttled
→ If consumed << provisioned: Overprovisioned (wasting money)
```

### Write Capacity
```
Same as read capacity.
```

### Latency
```
SuccessfulRequestLatency
→ Increasing: Table hot or overloaded
→ Stable: Healthy
```

### Errors
```
UserErrors: Bad query, invalid item, missing key
→ Fix application

SystemErrors: AWS issue
→ Usually temporary, retry

Throttling: Hit capacity limit
→ Increase provisioned RCU/WCU or use on-demand
```

---

## Setting Alarms

```go
// Alarm: Throttling
cloudwatch.PutMetricAlarm({
    AlarmName: "DynamoDB-Throttling",
    MetricName: "Throttling",
    Namespace: "AWS/DynamoDB",
    Statistic: "Sum",
    Period: 60,
    EvaluationPeriods: 1,
    Threshold: 1,
    ComparisonOperator: "GreaterThanOrEqualToThreshold",
    AlarmActions: ["arn:aws:sns:..."],
})

// Alarm: High Latency
cloudwatch.PutMetricAlarm({
    AlarmName: "DynamoDB-HighLatency",
    MetricName: "SuccessfulRequestLatency",
    Statistic: "Average",
    Threshold: 100,  // ms
    ComparisonOperator: "GreaterThanThreshold",
})
```

---

## CloudTrail Logging

Track API calls for compliance.

```
CreateTable
PutItem
DeleteItem
UpdateItem
Query
...

Logged with: User, timestamp, parameters, result
```

---

## X-Ray Tracing

Trace requests end-to-end.

```go
sess := session.New()
sess.Handlers.Build.PushBack(xray.AWS)

client := dynamodb.New(sess)
// All DynamoDB calls traced
```

---

## Application-Level Observability

```go
// Log all queries
logger.Info("Query executed",
    "table": "orders",
    "pk": "user#123",
    "duration_ms": 45,
    "items_returned": 10,
)

// Track slow queries
if duration > 100*time.Millisecond {
    logger.Warn("Slow query detected",
        "query": query,
        "duration_ms": duration,
    )
}

// Track errors
if err != nil {
    logger.Error("DynamoDB error",
        "error": err,
        "table": "orders",
    )
}
```

---

## Key Takeaway

Monitor:
- Consumed vs provisioned capacity
- Latency trends
- Error rates
- Throttling

Alert on:
- Throttling (capacity exceeded)
- High latency (performance degradation)
- System errors (AWS issues)

**Next:** [Common Pitfalls & Anti-Patterns](./16-anti-patterns.md)
