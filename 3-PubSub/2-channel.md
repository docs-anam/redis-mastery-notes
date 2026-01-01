# Redis Pub/Sub Channels

## Overview
Channels are the core routing mechanism in Redis Pub/Sub. They act as named topics through which publishers send messages and subscribers listen for them. Channels are dynamically created on first use and have no predetermined schema.

## Channel Characteristics

### Basic Properties
- **Named topics**: Use any string as a channel name
- **Dynamic creation**: Channels exist only when active
- **No persistence**: Channel data isn't stored
- **Case-sensitive**: "Orders" and "orders" are different channels
- **Unlimited channels**: Create as many as needed
- **Instant creation**: Created on first subscription/publish

### Channel Naming Conventions

Best practice is hierarchical naming with colons as separators:

```
service:entity:event
user:notifications:login
order:status:update
payment:error:insufficient_funds
alerts:system:cpu_high
cache:invalidation:product
```

### Benefits of Good Naming
```
✅ Hierarchical structure
✅ Easy pattern matching
✅ Self-documenting
✅ Easier debugging
✅ Logical organization
❌ Bad: "updates" (too generic)
❌ Bad: "u_n_e" (unclear abbreviations)
```

## Channel Operations

### Basic Channel Commands

| Command | Syntax | Complexity | Returns |
|---------|--------|-----------|---------|
| PUBLISH | PUBLISH channel message | O(N+M) | Number of subscribers |
| SUBSCRIBE | SUBSCRIBE channel [channel ...] | O(1) | Subscription confirmation |
| PSUBSCRIBE | PSUBSCRIBE pattern [pattern ...] | O(1) | Subscription confirmation |
| PUBSUB CHANNELS | PUBSUB CHANNELS [pattern] | O(N) | List of active channels |
| PUBSUB NUMSUB | PUBSUB NUMSUB channel [...] | O(N) | Subscriber count per channel |
| PUBSUB NUMPAT | PUBSUB NUMPAT | O(1) | Pattern subscription count |
| UNSUBSCRIBE | UNSUBSCRIBE channel [...] | O(1) | Unsubscribe confirmation |
| PUNSUBSCRIBE | PUNSUBSCRIBE pattern [...] | O(1) | Unsubscribe confirmation |

### Example: Check Active Channels
```redis
PUBSUB CHANNELS
// Returns: ["orders", "notifications", "alerts"]

PUBSUB CHANNELS order*
// Returns: ["orders", "order:status"]

PUBSUB NUMSUB orders notifications
// Returns: ["orders", 5, "notifications", 3]
```

## Pattern Subscriptions

### Wildcard Patterns
Subscribe to channel families using glob-style patterns:

| Pattern | Matches | Doesn't Match |
|---------|---------|---------------|
| `*` | All channels | Nothing |
| `news:*` | news:sports, news:tech | news, news:sports:live:score |
| `user:*:notify` | user:123:notify, user:456:notify | user:notify, user:123:notification |
| `alert:*:*` | alert:system:cpu, alert:db:memory | alert, alert:system |
| `*:error` | user:error, system:error | error, error:system |

### Pattern Matching Rules
- `*` matches any number of characters
- `?` matches exactly one character
- `[abc]` matches one character in the set
- `[a-z]` matches character ranges
- Use `\` to escape special characters

### Example: Pattern Subscriptions
```redis
PSUBSCRIBE news:*
// Subscribes to: news:sports, news:tech, news:health, etc.

PSUBSCRIBE user:*:notify
// Subscribes to: user:123:notify, user:456:notify, etc.

PSUBSCRIBE *:error:*
// Subscribes to: system:error:high, db:error:memory, etc.
```

## Channel Design Patterns

### 1. Hierarchical Topics
Create channel families for different entity types:

```
users:created
users:updated
users:deleted
orders:placed
orders:shipped
orders:delivered
payments:processed
payments:failed
```

### 2. Event-Driven Channels
Name channels after events that occur:

```
event:user:signup
event:user:login
event:user:logout
event:order:created
event:order:confirmed
event:payment:success
```

### 3. Service Communication Channels
Channels for inter-service communication:

```
service:auth:events
service:inventory:updates
service:payment:notifications
service:email:queue
```

### 4. Status Update Channels
For real-time status changes:

```
status:user:123:online
status:order:456:processing
status:stream:789:active
cache:invalidation:product:123
```

### 5. Notification Channels
User-specific and broadcast notifications:

```
user:123:notifications
user:456:notifications
broadcast:maintenance
broadcast:announcement
alerts:critical
```

## Advanced Channel Patterns

### Multi-Channel Subscription
Subscribe to multiple specific channels:

```python
import redis

pubsub = redis.Redis().pubsub()
pubsub.subscribe(
    'orders:placed',
    'orders:shipped',
    'orders:delivered'
)
```

### Pattern + Specific Subscriptions
Combine both types for flexibility:

```python
pubsub = redis.Redis().pubsub()
pubsub.subscribe('critical:alerts')  # Specific
pubsub.psubscribe('user:*:notifications')  # Pattern
```

### Dynamic Channel Management
Handle channels that change over time:

```python
pubsub = redis.Redis().pubsub()

def switch_user_channel(user_id):
    # Unsubscribe from previous
    pubsub.unsubscribe(f'user:{prev_id}:notifications')
    # Subscribe to new user
    pubsub.subscribe(f'user:{user_id}:notifications')
```

## Channel Management Strategies

### Monitoring Active Channels

```python
import redis

r = redis.Redis()

# Get all active channels
channels = r.pubsub_channels()
print(f"Active channels: {channels}")

# Get channels matching pattern
channels = r.pubsub_channels('user:*')
print(f"User channels: {channels}")

# Get subscriber count per channel
stats = r.pubsub_numsub('orders', 'notifications')
print(f"Subscriber stats: {stats}")

# Get pattern subscription count
pattern_count = r.pubsub_numpat()
print(f"Pattern subscriptions: {pattern_count}")
```

### Channel Organization by Service

```
# Payment Service
payment:invoice:created
payment:invoice:updated
payment:transaction:success
payment:transaction:failed

# Email Service
email:queue:process
email:notification:sent
email:error:retry

# Analytics Service
analytics:event:tracked
analytics:user:action
analytics:system:metric
```

## Message Publishing to Channels

### Simple Publishing
```python
r.publish('orders', 'order:12345:placed')
# Returns: number of subscribers that received it
```

### Publishing with JSON
```python
import json

data = {
    'order_id': 12345,
    'user_id': 789,
    'total': 99.99,
    'status': 'placed'
}

r.publish('orders', json.dumps(data))
```

### Publishing with Metadata
```python
import json
import time

message = {
    'event': 'order:placed',
    'data': {'order_id': 12345},
    'timestamp': time.time(),
    'source': 'order-service'
}

r.publish('events:orders', json.dumps(message))
```

## Channel Performance

### Publish Complexity Analysis

```
PUBLISH orders "message"
Time = O(N + M)
- N = Number of specific channel subscribers
- M = Number of pattern subscribers that match
```

### Scaling Considerations

| Aspect | Impact | Solution |
|--------|--------|----------|
| Many channels | Higher memory | Use pattern subscriptions |
| Many subscribers | Higher publish time | Optimize channel design |
| Many patterns | Higher matching time | Use specific subscriptions |
| Large messages | Memory overhead | Compression, ID references |

### Performance Tips

```python
# Good: Publish to specific channels
r.publish('order:placed', order_json)

# Avoid: Broadcasting to too many channels
for user_id in range(1000000):
    r.publish(f'user:{user_id}:update', data)  # Bad!

# Better: Use channel naming that allows pattern subscriptions
# Subscribers use: PSUBSCRIBE user:*:update
```

## Channel Best Practices

### 1. Use Hierarchical Naming
```
✅ orders:placed
✅ orders:shipped
❌ order_placed
❌ orderplaced
```

### 2. Keep Channel Names Reasonable
```
✅ "user:123:notifications"
❌ "u_1_2_3_n_o_t_i_f_i_c_a_t_i_o_n_s"
```

### 3. Document Your Channels
```python
# Keep a registry of channels
CHANNEL_REGISTRY = {
    'orders': 'Order status updates',
    'user:*:notifications': 'User-specific notifications',
    'alerts:critical': 'Critical system alerts'
}
```

### 4. Use Patterns for User-Specific Channels
```python
# Instead of subscribing to each user individually
pubsub.subscribe(f'user:{user_id}:messages')

# Use pattern to listen to all
pubsub.psubscribe('user:*:messages')
```

### 5. Monitor Channel Usage
```python
# Log active channels periodically
active_channels = r.pubsub_channels()
subscriber_counts = r.pubsub_numsub(*active_channels)

for channel, count in zip(active_channels, subscriber_counts):
    logger.info(f"Channel: {channel}, Subscribers: {count}")
```

## Common Channel Mistakes

### Mistake 1: Non-Deterministic Channel Names
```python
# Bad: Random channel names
channel = f"user:{user_id}:msg_{random.random()}"

# Good: Consistent naming
channel = f"user:{user_id}:messages"
```

### Mistake 2: Too Many Channels
```python
# Bad: One channel per unique combination
for order_id in all_orders:
    for user_id in all_users:
        publish(f"order:{order_id}:user:{user_id}", data)

# Good: Use hierarchical channels
publish("orders:updates", data)
```

### Mistake 3: Ignoring Pattern Performance
```python
# Bad: Many patterns
pubsub.psubscribe('*')  # Matches everything
pubsub.psubscribe('a*', 'b*', 'c*', 'd*', ...)

# Good: Use specific patterns
pubsub.psubscribe('orders:*', 'notifications:*')
```

### Mistake 4: No Channel Registry
```python
# Bad: Channels scattered across code
r.publish('x', 'y')
r.publish('abc', 'def')

# Good: Centralized channel definitions
class Channels:
    ORDERS = 'orders:updates'
    NOTIFICATIONS = 'user:*:notifications'
```

## Advanced Use Cases

### 1. Broadcast to Multiple Related Channels
```python
def broadcast_order_update(order_id, status):
    message = json.dumps({'order_id': order_id, 'status': status})
    
    # Notify different subscribers
    r.publish(f'orders:{status}', message)
    r.publish('orders:all', message)
    r.publish(f'order:{order_id}', message)
```

### 2. Topic-Based Message Filtering
```python
# Publisher tags messages by topic
message = {
    'topic': 'billing',
    'event': 'invoice_generated',
    'data': {...}
}
r.publish('events:all', json.dumps(message))

# Subscribers can filter by topic
for message in pubsub.listen():
    if message['type'] == 'message':
        event = json.loads(message['data'])
        if event['topic'] == 'billing':
            process_billing_event(event)
```

### 3. Channel Hierarchy Traversal
```python
def publish_hierarchical(category, action, data):
    message = json.dumps(data)
    
    # Publish to multiple levels
    r.publish(f'{category}:{action}', message)
    r.publish(f'{category}:all', message)
    r.publish('all:events', message)
```

## Channel Monitoring Commands

```python
# List all active channels
r.pubsub_channels()

# Get channels matching pattern
r.pubsub_channels('order:*')

# Get subscriber count for channels
r.pubsub_numsub('orders', 'notifications')

# Get total pattern subscriptions
r.pubsub_numpat()
```

## Summary

Channels are the fundamental routing mechanism in Redis Pub/Sub:
- Use hierarchical, meaningful names
- Combine specific and pattern subscriptions strategically
- Monitor channel usage for performance
- Document your channel architecture
- Design channels around your domain events and entities
