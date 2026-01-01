# Redis Streams - Event Logs & Message Queues

## Overview

Redis Streams are append-only logs with consumer groups. Perfect for:
- **Event Logs**: Audit trails, transaction logs
- **Message Queues**: Distributed messaging
- **Time-Series Data**: Sensor data, metrics
- **Activity Feeds**: Real-time notifications
- **Change Data Capture**: Event sourcing

### Why Streams?

- **Append-only**: Ordered by time, immutable
- **Consumer Groups**: Distribute messages across workers
- **Acknowledgment**: Track which messages are processed
- **Auto-ID**: Unique IDs assigned automatically
- **Replay**: Consume from any point in history

---

## Core Commands

### Add Entries

```bash
# Add entry with auto ID
XADD mystream * field1 value1 field2 value2
# Returns: 1234567890000-0

# Add with specific ID
XADD mystream 1000-0 field1 value1

# Get stream length
XLEN mystream
```

### Read Entries

```bash
# Read range
XRANGE mystream - +                     # All entries
XRANGE mystream 1000-0 2000-0          # Within ID range

# Read latest
XREVRANGE mystream + -                  # Newest first
XREVRANGE mystream + - COUNT 10        # Last 10
```

### Consumer Groups

```bash
# Create consumer group
XGROUP CREATE mystream mygroup 0        # From start

# Read as consumer
XREADGROUP GROUP mygroup consumer1 STREAMS mystream >

# Acknowledge message
XACK mystream mygroup message-id

# Pending messages
XPENDING mystream mygroup
```

### Trim & Delete

```bash
# Trim stream (keep N recent)
XTRIM mystream MAXLEN 1000

# Delete entry
XDEL mystream message-id
```

---

## Practical Examples

### Example 1: Event Log

```python
import redis
import time

r = redis.Redis()

# Log event
event = {
    'user_id': '123',
    'action': 'login',
    'ip': '192.168.1.1',
    'timestamp': str(time.time())
}
r.xadd('events', '*', **event)

# Get recent events
events = r.xrevrange('events', count=10)
for event_id, event_data in events:
    print(f"{event_id}: {event_data}")
```

### Example 2: Distributed Task Queue

```python
# Producer: Add tasks
task = {
    'type': 'send_email',
    'user_id': '123',
    'email': 'user@example.com'
}
r.xadd('tasks', '*', **task)

# Consumer group: Multiple workers
r.xgroup_create('tasks', 'workers', id='0', mkstream=True)

# Worker: Process tasks
def worker(worker_id):
    while True:
        messages = r.xreadgroup(
            'workers', 
            worker_id,
            {'tasks': '>'},
            count=1
        )
        
        for stream_name, msg_list in messages:
            for msg_id, data in msg_list:
                try:
                    process_task(data)
                    r.xack('tasks', 'workers', msg_id)
                except Exception as e:
                    log.error(f"Failed: {e}")
```

### Example 3: Time-Series Data

```python
# Record metrics
metrics = {
    'cpu': '45.2',
    'memory': '62.1',
    'disk': '78.5'
}
r.xadd('metrics', '*', **metrics)

# Get metrics from last hour
one_hour_ago = int((time.time() - 3600) * 1000)
recent_metrics = r.xrange('metrics', one_hour_ago)

# Calculate averages
cpu_values = [float(m[1][b'cpu']) for m in recent_metrics]
avg_cpu = sum(cpu_values) / len(cpu_values)
```

---

## Real-World Patterns

### Pattern 1: Audit Trail

```python
def log_action(user_id, action, details):
    """Log user action"""
    event = {
        'user_id': str(user_id),
        'action': action,
        'details': json.dumps(details),
        'timestamp': str(time.time())
    }
    r.xadd('audit:trail', '*', **event)

def get_user_activity(user_id, limit=100):
    """Get user's recent activities"""
    all_events = r.xrevrange('audit:trail', count=limit*10)
    
    user_events = [
        (eid, data) for eid, data in all_events
        if data[b'user_id'].decode() == str(user_id)
    ]
    
    return user_events[:limit]
```

### Pattern 2: Consumer Groups with Retry

```python
def process_stream_with_retry():
    """Process messages with retry logic"""
    stream_key = 'orders'
    group_name = 'processors'
    consumer_name = f'worker-{os.getpid()}'
    
    # Create group if not exists
    try:
        r.xgroup_create(stream_key, group_name, id='0', mkstream=True)
    except redis.ResponseError:
        pass  # Group already exists
    
    while True:
        # Get pending messages first (failed previously)
        pending = r.xpending(stream_key, group_name)
        
        if pending['pending'] > 0:
            # Retry failed messages
            messages = r.xclaim(
                stream_key,
                group_name,
                consumer_name,
                min_idle_time=60000,  # 1 minute
                count=10
            )
        else:
            # Get new messages
            messages = r.xreadgroup(
                group_name,
                consumer_name,
                {stream_key: '>'},
                count=10
            )
        
        for stream, msg_list in messages:
            for msg_id, data in msg_list:
                try:
                    process_order(data)
                    r.xack(stream_key, group_name, msg_id)
                except Exception as e:
                    log.error(f"Failed {msg_id}: {e}")
```

---

## Performance

```
Operation              Time         Memory
──────────────────────────────────────────
XADD                  O(1)         30 bytes per entry
XREAD                 O(N)         Returns N entries
XRANGE                O(log N + N) For N entries
XLEN                  O(1)         Constant
XDEL                  O(N)         For N entries
```

---

## Best Practices

### 1. Use Consumer Groups for Distributed Processing

```python
# ✅ DO: Consumer groups for distribution
r.xgroup_create('tasks', 'workers', id='0', mkstream=True)
messages = r.xreadgroup('workers', worker_id, {'tasks': '>'})

# ❌ DON'T: Manual consumer tracking
tasks = r.xread({'tasks': last_id}, count=10)
# No automatic load balancing!
```

### 2. Acknowledge Processed Messages

```python
# ✅ DO: ACK after successful processing
try:
    process(data)
    r.xack(stream, group, msg_id)
except Exception:
    # Don't ACK - will be retried
    pass

# ❌ DON'T: ACK before processing
r.xack(stream, group, msg_id)
process(data)  # If fails here, message is lost!
```

### 3. Monitor Pending Messages

```python
# ✅ DO: Check for stuck messages
pending = r.xpending(stream, group)
if pending['pending'] > 100:
    alert("Too many pending messages!")

# Reclaim stuck messages
r.xclaim(stream, group, consumer, min_idle_time=3600000)
```

### 4. Trim Streams Periodically

```python
# ✅ DO: Keep streams bounded
# Run daily cleanup
r.xtrim('events', maxlen=1000000)

# ✅ DO: Archive old entries
old_events = r.xrange('events', count=10000)
save_to_database(old_events)
r.xtrim('events', maxlen=100000)
```

---

## Common Mistakes

### Mistake 1: No Consumer Group Management

```python
# ❌ WRONG: No acknowledgment tracking
entry = r.xread({'stream': last_id}, count=1)
process(entry)  # If crashes, message might be lost

# ✅ RIGHT: Use consumer groups
r.xreadgroup(group, consumer, {'stream': '>'})
# Automatic tracking, can retry if needed
```

### Mistake 2: Unbounded Stream Growth

```python
# ❌ WRONG: Never trimmed
r.xadd('events', '*', ...)
# After years: billion entries!

# ✅ RIGHT: Trim regularly
r.xtrim('events', maxlen=1000000)  # Keep 1M latest
```

### Mistake 3: No Error Handling

```python
# ❌ WRONG: ACK before processing
r.xack(stream, group, msg_id)
process(data)
# If process fails, message is already ACKed!

# ✅ RIGHT: ACK after success
process(data)
r.xack(stream, group, msg_id)
```

### Mistake 4: Ignoring Consumer Lag

```python
# ❌ WRONG: Don't monitor lag
# No visibility into processing speed

# ✅ RIGHT: Monitor consumer group info
info = r.xinfo_groups(stream)
for group in info:
    print(f"Pending: {group['pending']}")
    print(f"Consumers: {group['consumers']}")
```

---

## Next Steps

- **[Sorted Sets](5-sorted-set.md)** - Rankings
- **[Hashes](4-hashes.md)** - Objects
- Advanced: Lua scripts with Streams, consumer group patterns

