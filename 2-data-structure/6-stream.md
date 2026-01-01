# Redis Streams

## Overview
Redis Streams is an append-only log data structure designed for high-throughput event streaming and message queue use cases (Redis 5.0+). Streams maintain entry sequence with automatic timestamping, similar to Apache Kafka, but with Redis simplicity.

## Key Characteristics
- **Append-only log**: New entries added sequentially, existing entries immutable
- **Auto-timestamped**: Each entry assigned unique timestamp-based ID
- **Ordered**: Entries maintain exact insertion order
- **Consumer Groups**: Multiple consumers can process same stream independently
- **Pending Entries**: Tracks unacknowledged messages per consumer
- **Range queries**: Efficient retrieval by ID or timestamp range

## Common Commands

### Adding Data
| Command | Complexity | Description |
|---------|-----------|-------------|
| `XADD key [ID] field value [...]` | O(1) | Append entry to stream |
| `XLEN key` | O(1) | Get stream length |
| `XTRIM key MAXLEN ~ count` | O(N) | Trim stream to max length |
| `XDEL key ID [ID ...]` | O(N) | Remove entries by ID |

### Reading Data
| Command | Complexity | Description |
|---------|-----------|-------------|
| `XRANGE key start end [COUNT]` | O(N) | Read entries by ID range |
| `XREVRANGE key end start [COUNT]` | O(N) | Read in reverse |
| `XREAD [BLOCK ms] STREAMS key ID` | O(N) | Read entries, optionally blocking |
| `XLEN key` | O(1) | Get stream length |

### Consumer Groups
| Command | Complexity | Description |
|---------|-----------|-------------|
| `XGROUP CREATE key group ID` | O(1) | Create consumer group |
| `XREADGROUP GROUP group consumer [STREAMS key ID]` | O(N) | Read as group member |
| `XACK key group ID [ID ...]` | O(N) | Acknowledge message |
| `XPENDING key group` | O(N) | List pending messages |
| `XCLAIM key group consumer min-idle-time ID` | O(N) | Claim pending message |

## Use Cases

### 1. Event Log & Audit Trail
Store all application events chronologically.

```redis
// Add events
XADD events:app * user_id 123 action "login" ip "192.168.1.1"
XADD events:app * user_id 123 action "viewed_product" product_id 456
XADD events:app * user_id 123 action "added_to_cart" product_id 456
XADD events:app * user_id 123 action "checkout" total 99.99

// Read all events
XRANGE events:app - +

// Read last 10 events
XREVRANGE events:app + - COUNT 10

// Get stream length
XLEN events:app

// Read events after specific time
XRANGE events:app (1705318200 +
```

**Real-world scenario**: Event tracking, user activity logs, audit trails.

### 2. Real-time Message Queue
Distribute messages to multiple consumers.

```redis
// Create consumer group
XGROUP CREATE messages mygroup $

// Producer adds messages
XADD messages * type "email" to "user@example.com" subject "Welcome"
XADD messages * type "sms" to "+1234567890" content "Hello"
XADD messages * type "push" user_id 123 title "New notification"

// Worker reads messages
XREADGROUP GROUP mygroup worker COUNT 1 STREAMS messages >

// Acknowledge processing
XACK messages mygroup 1705318200-0

// Get pending messages
XPENDING messages mygroup

// Claim abandoned message
XCLAIM messages mygroup worker 3600000 1705318200-0
```

**Real-world scenario**: Distributed message queues, job processing, notification systems.

### 3. Time-series Data Collection
Collect measurements/metrics over time.

```redis
// Collect temperature readings
XADD temperature:sensor:1 * value 23.5 humidity 65 timestamp "2024-01-15T10:00:00Z"
XADD temperature:sensor:1 * value 23.8 humidity 64 timestamp "2024-01-15T10:05:00Z"
XADD temperature:sensor:1 * value 24.2 humidity 63 timestamp "2024-01-15T10:10:00Z"

// Query readings from today
XRANGE temperature:sensor:1 1705318200 1705404600

// Keep only last day
XTRIM temperature:sensor:1 MAXLEN ~ 17280  // ~17280 entries per day (5min intervals)

// Analyze data
XLEN temperature:sensor:1  // Total readings
```

**Real-world scenario**: IoT data collection, metrics, sensor data, application monitoring.

### 4. Activity Feed (Social Media)
Store user activities for followers to see.

```redis
// User posts activity
XADD user:123:feed * action "post" content "Great day today!" timestamp "2024-01-15T10:30:00Z"
XADD user:123:feed * action "like" liked_post_id 456 timestamp "2024-01-15T10:35:00Z"
XADD user:123:feed * action "follow" followed_user 789 timestamp "2024-01-15T10:40:00Z"

// Get user's recent activities
XREVRANGE user:123:feed + - COUNT 50

// Get activities from followers
XRANGE user:123:feed (1705318000 +
```

**Real-world scenario**: Social media feeds, activity streams, user timelines.

### 5. Distributed Work Queue with Acknowledgment
Ensure tasks are processed exactly once.

```redis
// Create group
XGROUP CREATE tasks:queue workers 0

// Add tasks
XADD tasks:queue * type "process_payment" order_id 12345 amount 99.99
XADD tasks:queue * type "send_email" user_id 123 template "order_confirmation"

// Worker gets tasks
XREADGROUP GROUP workers worker COUNT 5 BLOCK 1000 STREAMS tasks:queue >

// Process task...
// Acknowledge when complete
XACK tasks:queue workers 1705318200-0

// Check pending
XPENDING tasks:queue workers
```

**Real-world scenario**: Distributed job processing, reliable message delivery.

## Advanced Patterns

### Consumer Group with Dead Letter Queue
```redis
// Create main and dead letter queues
XGROUP CREATE events main $
XGROUP CREATE events:dlq handlers 0

// Read from main
XREADGROUP GROUP main handler COUNT 1 STREAMS events >

// If processing fails, move to DLQ
XADD events:dlq * original_id <id> reason "processing_failed"
```

### Stream Aggregation
```redis
// Multiple streams (sensors)
XADD sensor:1 * temperature 20
XADD sensor:2 * temperature 22
XADD sensor:3 * temperature 21

// Read from all sensors
XREAD STREAMS sensor:1 sensor:2 sensor:3 0 0
```

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Add entry | O(1) | Append operation |
| Read range | O(N) | N = entries returned |
| Stream length | O(1) | Cached length |
| Trim | O(N) | N = entries removed |
| Group operations | O(N) | Varies by operation |

## Memory Considerations

- Stream entries consume ~100-200 bytes minimum
- Consumer groups add ~50-100 bytes per entry per group
- Pending entries tracked separately (~50 bytes each)
- Consider XTRIM for unbounded streams

## Limitations

- **No deduplication**: Duplicate entries allowed, deduplicate in application
- **No sharding**: Single stream on one node
- **Immutable entries**: Cannot modify, only delete
- **ID constraints**: Can't backdate entries (timestamp-based IDs)
- **Consumer group state**: Stored in memory, lost on restart

## Best Practices

1. **Set max stream length**: Use XTRIM to prevent unbounded growth
2. **Use consumer groups**: For reliable, distributed processing
3. **Handle pending entries**: Monitor with XPENDING, reclaim with XCLAIM
4. **Autoincrement IDs**: Use * in XADD for automatic ID generation
5. **Batch reads**: Use COUNT to limit entries per read
6. **Set appropriate block time**: Balance latency vs CPU usage
7. **Monitor group lag**: Track difference between latest ID and consumer position

## Common Pitfalls

1. **Unbounded streams**: Forgetting XTRIM causes memory issues
2. **Consumer lag**: Not reading fast enough from long streams
3. **Unacknowledged messages**: Pending entries pile up, causing memory issues
4. **Blocking indefinitely**: XREAD BLOCK 0 can cause hanging
5. **Lost consumer state**: Consumer groups don't persist across restart
6. **Wrong ID format**: Entry IDs must be timestamp-based format
