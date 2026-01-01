# Pub/Sub - Real-Time Message Broadcasting

## Table of Contents
1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Commands Reference](#commands-reference)
4. [Practical Examples](#practical-examples)
5. [Real-World Patterns](#real-world-patterns)
6. [Performance Characteristics](#performance-characteristics)
7. [Best Practices](#best-practices)
8. [Common Mistakes](#common-mistakes)

---

## Overview

Pub/Sub (Publish/Subscribe) provides a message delivery system where:
- **Publishers** send messages to channels
- **Subscribers** receive messages from channels
- **Decoupling**: Publishers and subscribers don't know about each other
- **Real-time**: Messages delivered immediately to all subscribers

### When to Use Pub/Sub

✅ **Good For**:
- Chat applications and messaging
- Real-time notifications (alerts, updates)
- Live data feeds (scores, stock prices)
- Cache invalidation signals
- Event broadcasting
- Real-time analytics

❌ **Not Good For**:
- Guaranteed message delivery (use Streams instead)
- Message persistence (use Streams instead)
- Message queuing (use Lists or Streams instead)
- Critical business data
- When you need replay capability

---

## Core Concepts

### Channels
Named communication pipes for messages.

```
Publisher -> Channel "alerts" -> Subscriber1
                              -> Subscriber2
                              -> Subscriber3
```

### Patterns
Wildcard subscriptions using glob patterns.

```
Pattern             Matches
────────────────────────────────
news:*             news:sports, news:tech, news:weather
user:*:alerts      user:123:alerts, user:456:alerts
*.notification     email.notification, sms.notification
```

### Limitations
Understanding Pub/Sub constraints:
- **No persistence**: Messages not stored
- **No message queue**: Missed messages are lost
- **No ack**: No confirmation subscribers received
- **No replay**: Can't get old messages
- **At most once**: No delivery guarantees

---

## Commands Reference

### PUBLISH
Broadcast message to a channel.

```redis
PUBLISH channel message
# Returns: Number of subscribers that received the message
```

```python
import redis

r = redis.Redis(host='localhost', port=6379)

# Publish message
num_subscribers = r.publish('alerts', 'New login detected')
print(f"Delivered to {num_subscribers} subscribers")
```

### SUBSCRIBE
Listen to one or more channels.

```redis
SUBSCRIBE channel [channel ...]
# Enters subscription mode - blocking call
```

```python
import redis

r = redis.Redis(host='localhost', port=6379)
pubsub = r.pubsub()

# Subscribe to channels
pubsub.subscribe('alerts', 'notifications')

# Listen for messages
for message in pubsub.listen():
    if message['type'] == 'message':
        print(f"Channel: {message['channel']}")
        print(f"Message: {message['data']}")
```

### PSUBSCRIBE
Subscribe using pattern matching.

```redis
PSUBSCRIBE pattern [pattern ...]
# Receives messages from matching channels
```

```python
import redis

r = redis.Redis(host='localhost', port=6379)
pubsub = r.pubsub()

# Subscribe to pattern
pubsub.psubscribe('user:*:alerts')

# Listen for messages
for message in pubsub.listen():
    if message['type'] == 'pmessage':
        print(f"Pattern: {message['pattern']}")
        print(f"Channel: {message['channel']}")
        print(f"Message: {message['data']}")
```

### UNSUBSCRIBE
Stop listening to channels.

```redis
UNSUBSCRIBE [channel ...]
# Unsubscribe from channels (omit args = unsubscribe all)
```

```python
pubsub.unsubscribe('alerts')
pubsub.punsubscribe('user:*:alerts')
```

### PUBSUB CHANNELS
List active channels.

```redis
PUBSUB CHANNELS [pattern]
# Returns: List of channels with subscribers
```

```python
# Get all channels with subscribers
channels = r.pubsub_channels()
print(f"Active channels: {channels}")

# Get matching channels
pattern_channels = r.pubsub_channels(pattern='user:*')
print(f"User channels: {pattern_channels}")
```

### PUBSUB NUMSUB
Get number of subscribers per channel.

```redis
PUBSUB NUMSUB channel [channel ...]
# Returns: List of [channel, count] pairs
```

```python
# Get subscriber count
result = r.execute_command('PUBSUB', 'NUMSUB', 'alerts', 'notifications')
print(result)
# Output: [b'alerts', 5, b'notifications', 3]
```

---

## Practical Examples

### Example 1: Real-Time Notifications

```python
import redis
import time
from threading import Thread

class NotificationSystem:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
    
    def subscribe_user(self, user_id):
        """Subscribe user to their notification channel"""
        pubsub = self.r.pubsub()
        channel = f'user:{user_id}:notifications'
        pubsub.subscribe(channel)
        
        print(f"User {user_id} subscribed to {channel}")
        
        for message in pubsub.listen():
            if message['type'] == 'message':
                notification = message['data'].decode('utf-8')
                print(f"📬 Notification: {notification}")
    
    def notify_user(self, user_id, message):
        """Send notification to user"""
        channel = f'user:{user_id}:notifications'
        num_delivered = self.r.publish(channel, message)
        print(f"Message delivered to {num_delivered} subscriber(s)")

# Usage
notif = NotificationSystem()

# Start listener in background
listener = Thread(target=notif.subscribe_user, args=(123,))
listener.daemon = True
listener.start()

time.sleep(1)  # Wait for subscription
notif.notify_user(123, "Your order has been shipped!")
notif.notify_user(123, "New comment on your post!")

time.sleep(2)
```

### Example 2: Chat Room

```python
import redis
from threading import Thread
import json
from datetime import datetime

class ChatRoom:
    def __init__(self, room_name):
        self.r = redis.Redis(host='localhost', port=6379)
        self.room = f'chat:{room_name}'
    
    def join(self, username):
        """Join chat room"""
        pubsub = self.r.pubsub()
        pubsub.subscribe(self.room)
        
        # Announce join
        self.broadcast(f"{username} joined the chat")
        
        # Listen for messages
        for message in pubsub.listen():
            if message['type'] == 'message':
                data = json.loads(message['data'])
                sender = data['user']
                text = data['text']
                print(f"{sender}: {text}")
    
    def send_message(self, username, text):
        """Send message to room"""
        message = {
            'user': username,
            'text': text,
            'timestamp': datetime.now().isoformat()
        }
        self.r.publish(self.room, json.dumps(message))
    
    def broadcast(self, system_message):
        """Send system message"""
        self.r.publish(self.room, json.dumps({
            'user': 'SYSTEM',
            'text': system_message
        }))

# Usage
room = ChatRoom('general')

# User 1 joins
user1 = Thread(target=room.join, args=('Alice',))
user1.daemon = True
user1.start()

import time
time.sleep(1)

# User 2 sends message
room.send_message('Bob', 'Hello everyone!')
room.send_message('Bob', 'How is everyone?')

time.sleep(2)
```

### Example 3: Cache Invalidation Broadcast

```python
import redis
from datetime import datetime, timedelta

class CachedDataService:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
    
    def get_user_data(self, user_id):
        """Get user data with caching"""
        cache_key = f'user:{user_id}'
        
        # Try cache first
        cached = self.r.get(cache_key)
        if cached:
            return cached.decode('utf-8')
        
        # Fetch from database (simulated)
        user_data = f"User {user_id} data"
        
        # Cache for 1 hour
        self.r.setex(cache_key, 3600, user_data)
        return user_data
    
    def update_user_data(self, user_id, new_data):
        """Update data and invalidate cache"""
        # Update database (simulated)
        
        # Invalidate cache
        cache_key = f'user:{user_id}'
        self.r.delete(cache_key)
        
        # Broadcast invalidation
        self.r.publish(f'cache:invalidate', cache_key)
        print(f"Cache invalidated: {cache_key}")
    
    def listen_for_invalidation(self):
        """Listen for cache invalidation events"""
        pubsub = self.r.pubsub()
        pubsub.subscribe('cache:invalidate')
        
        for message in pubsub.listen():
            if message['type'] == 'message':
                cache_key = message['data'].decode('utf-8')
                print(f"Cache invalidation received: {cache_key}")
                # Handle invalidation (clear local cache, etc.)

# Usage
service = CachedDataService()

# Get data (cached)
data = service.get_user_data(123)
print(data)

# Update data (invalidates cache)
service.update_user_data(123, "Updated data")
```

### Example 4: Real-Time Leaderboard Updates

```python
import redis
import json
from threading import Thread

class LeaderboardBroadcaster:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
    
    def update_score(self, player_id, score_delta):
        """Update player score and broadcast"""
        leaderboard_key = 'game:leaderboard'
        
        # Update score
        new_score = self.r.zincrby(leaderboard_key, score_delta, player_id)
        
        # Get updated rank
        rank = self.r.zrevrank(leaderboard_key, player_id) + 1
        
        # Broadcast update
        update = {
            'player_id': player_id,
            'score': float(new_score),
            'rank': rank,
            'delta': score_delta
        }
        self.r.publish('leaderboard:updates', json.dumps(update))
    
    def subscribe_to_updates(self):
        """Listen for leaderboard updates"""
        pubsub = self.r.pubsub()
        pubsub.subscribe('leaderboard:updates')
        
        for message in pubsub.listen():
            if message['type'] == 'message':
                update = json.loads(message['data'])
                print(f"🎮 Player {update['player_id']} now has "
                      f"{update['score']} points (Rank: {update['rank']})")
    
    def get_leaderboard(self, limit=10):
        """Get top players"""
        top_players = self.r.zrevrange('game:leaderboard', 0, limit-1, 
                                       withscores=True)
        return [(p.decode(), int(s)) for p, s in top_players]

# Usage
broadcaster = LeaderboardBroadcaster()

# Start listener
listener = Thread(target=broadcaster.subscribe_to_updates)
listener.daemon = True
listener.start()

import time
time.sleep(0.5)

# Update scores
broadcaster.update_score('player_1', 100)
broadcaster.update_score('player_2', 150)
broadcaster.update_score('player_1', 50)

# Get leaderboard
print(broadcaster.get_leaderboard())

time.sleep(2)
```

---

## Real-World Patterns

### Pattern 1: Fan-Out Broadcasting

Broadcasting to multiple subscribers efficiently:

```python
import redis
from threading import Thread
import time

class EventBroadcaster:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
    
    def broadcast_event(self, event_type, data):
        """Broadcast event to all listeners"""
        message = f"{event_type}:{data}"
        
        # Single publish reaches all subscribers
        num_subscribers = self.r.publish('events', message)
        print(f"Event broadcast to {num_subscribers} subscribers")
    
    def subscribe(self, name):
        """Subscribe to events"""
        pubsub = self.r.pubsub()
        pubsub.subscribe('events')
        
        print(f"Subscriber {name} listening...")
        for message in pubsub.listen():
            if message['type'] == 'message':
                print(f"[{name}] Received: {message['data'].decode()}")

# Usage
broadcaster = EventBroadcaster()

# Start multiple subscribers
for i in range(3):
    Thread(target=broadcaster.subscribe, args=(f'Sub_{i}',), daemon=True).start()

time.sleep(1)

# Broadcast to all
broadcaster.broadcast_event('order', 'Order #123 created')
broadcaster.broadcast_event('payment', 'Payment received')

time.sleep(2)
```

### Pattern 2: Multi-Level Topic Hierarchy

Organizing messages with pattern matching:

```python
import redis

class HierarchicalMessaging:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
    
    def publish(self, topic, message):
        """Publish to specific topic"""
        self.r.publish(topic, message)
    
    def subscribe_pattern(self, pattern, callback):
        """Subscribe to pattern"""
        pubsub = self.r.pubsub()
        pubsub.psubscribe(pattern)
        
        for message in pubsub.listen():
            if message['type'] == 'pmessage':
                callback(message['channel'], message['data'])

# Usage
messaging = HierarchicalMessaging()

def handle_alert(channel, data):
    print(f"Alert on {channel}: {data.decode()}")

# Listen to all user alerts
messaging.subscribe_pattern('user:*:alerts', handle_alert)

# Publish to different users
messaging.publish('user:123:alerts', 'Login detected')
messaging.publish('user:456:alerts', 'Password changed')
```

---

## Performance Characteristics

### Throughput

```
Scenario                  Throughput        Latency
───────────────────────────────────────────────────
Single message            ~1M msg/sec      <1ms
Broadcast to 10 subs      ~100k msg/sec    1-5ms
Broadcast to 100 subs     ~50k msg/sec     5-10ms
Broadcast to 1000 subs    ~10k msg/sec     10-50ms
```

### Memory Usage
- **Per subscription**: ~100-500 bytes
- **10,000 subscriptions**: ~1-5 MB
- **No message buffering**: Constant memory

### Scalability Limits

```
Configuration          Max Subscribers    Notes
──────────────────────────────────────────────
Single Redis           10,000+           Limited by network
Redis Cluster          1M+               Distributed
Load Balancer + Redis  100M+             Multiple Redis nodes
```

---

## Best Practices

### DO: Use Channels Wisely

```python
# ✅ GOOD: Hierarchical channels
PUBLISH 'notifications:email:marketing', message
PUBLISH 'notifications:sms:alerts', message
PUBLISH 'notifications:push:critical', message

# ✅ GOOD: Topic-based separation
PUBLISH 'orders:created', message
PUBLISH 'orders:shipped', message
PUBLISH 'orders:delivered', message
```

### DO: Handle Disconnections

```python
import redis
import time

def robust_subscribe(channel):
    """Subscribe with reconnection handling"""
    r = redis.Redis(host='localhost', port=6379)
    pubsub = r.pubsub()
    
    while True:
        try:
            pubsub.subscribe(channel)
            for message in pubsub.listen():
                if message['type'] == 'message':
                    print(f"Received: {message['data']}")
        except redis.ConnectionError:
            print("Connection lost, reconnecting...")
            time.sleep(5)
            pubsub = r.pubsub()
```

### DO: Use Patterns for Flexibility

```python
# ✅ GOOD: Pattern subscriptions
pubsub.psubscribe('alerts:*:critical')
pubsub.psubscribe('user:*:updates')
pubsub.psubscribe('system:*:errors')

# Receives messages like:
# alerts:database:critical
# alerts:api:critical
# user:123:updates
# user:456:updates
```

### DON'T: Rely on Message Delivery

```python
# ❌ BAD: Assuming Pub/Sub guarantees delivery
# If no one is subscribed, message is lost!
self.r.publish('critical_alert', data)

# ✅ GOOD: Use Streams for reliability
self.r.xadd('critical_alerts', {'data': data})
```

### DON'T: Block Subscribers

```python
# ❌ BAD: Long processing in subscriber
for message in pubsub.listen():
    process_expensive_operation(message)  # Blocks listening

# ✅ GOOD: Queue for processing
for message in pubsub.listen():
    task_queue.put(message)  # Immediate return
    # Process in thread pool elsewhere
```

### DON'T: Send Large Payloads

```python
# ❌ BAD: Sending large data
large_data = json.dumps(huge_object)
self.r.publish('data', large_data)

# ✅ GOOD: Send reference, store separately
key = f'data:{uuid.uuid4()}'
self.r.setex(key, 300, json.dumps(huge_object))
self.r.publish('data', key)
```

---

## Common Mistakes

### Mistake 1: Assuming Guaranteed Delivery

**Problem**: Messages are lost if no subscribers are listening
```python
# ❌ This message might be lost!
self.r.publish('alerts', 'Server down!')

if no_one_listening:
    # Message disappeared
```

**Solution**: Use Streams for guaranteed delivery
```python
# ✅ Message persisted
self.r.xadd('alerts', {'message': 'Server down!'})
```

### Mistake 2: Publishing Before Subscribing

**Problem**: Subscriber misses messages published before connecting
```python
# ❌ Race condition
self.r.publish('ready', 'System started')

# Now subscriber connects
pubsub.subscribe('ready')
# Already missed the message
```

**Solution**: Ensure subscribers are ready first, or use persistent queue
```python
# ✅ Use Streams for persistence
self.r.xadd('events', {'type': 'system_started'})
```

### Mistake 3: No Connection Handling

**Problem**: Subscriber crashes if connection drops
```python
# ❌ Will fail on disconnection
for message in pubsub.listen():
    process(message)
```

**Solution**: Add reconnection logic
```python
# ✅ Reconnects automatically
while True:
    try:
        for message in pubsub.listen():
            process(message)
    except redis.ConnectionError:
        pubsub = r.pubsub()
```

### Mistake 4: Blocking Subscribers

**Problem**: Slow processing blocks message reception
```python
# ❌ Slow processing blocks listening
for message in pubsub.listen():
    slow_database_operation(message)
```

**Solution**: Queue messages for async processing
```python
# ✅ Non-blocking reception
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=5)
for message in pubsub.listen():
    executor.submit(process_message, message)
```

### Mistake 5: Too Many Patterns

**Problem**: Patterns degrade performance
```python
# ❌ Pattern matching is O(N) - avoid too many
pubsub.psubscribe('user:*:*:*:*:*:*')
pubsub.psubscribe('orders:*:*:*:*')
pubsub.psubscribe('products:*:*:*:*')
```

**Solution**: Use fixed channels when possible
```python
# ✅ Direct subscriptions are O(1)
pubsub.subscribe('user_updates')
pubsub.subscribe('order_updates')
pubsub.subscribe('product_updates')
```

### Mistake 6: Not Monitoring Subscriber Count

**Problem**: No visibility into subscriber health
```python
# ❌ Publishing to ghost channels
self.r.publish('alerts', message)
# No one listening but no indication
```

**Solution**: Monitor active subscribers
```python
# ✅ Check before publishing
def publish_if_subscribers(channel, message):
    count = self.r.execute_command('PUBSUB', 'NUMSUB', channel)[1]
    if count > 0:
        self.r.publish(channel, message)
    else:
        log.warning(f"No subscribers on {channel}")
```

---

## When to Use Alternatives

| Use Case | Pub/Sub | Streams | Lists |
|----------|---------|---------|-------|
| Real-time notifications | ✅ | ❌ | ❌ |
| Persistent messages | ❌ | ✅ | ✅ |
| Message queue | ❌ | ✅ | ✅ |
| Fan-out broadcasting | ✅ | ⚠️ | ❌ |
| Message replay | ❌ | ✅ | ❌ |
| Multiple consumers | ✅ | ✅ | ❌ |

---

## Next Steps

1. **Explore [Streams](../2-data-structure/6-stream.md)** for persistent messaging
2. **Learn [Scripting](3-scripting.md)** for complex event handling
3. **Study [Replication](4-replication.md)** for distributed Pub/Sub
4. **Set up [Monitoring](../1-basics/11-server-information.md)** for production systems

## Resources

- [Redis Pub/Sub Documentation](https://redis.io/topics/pubsub)
- [Pub/Sub Performance](https://redis.io/topics/pubsub#performance)
- [Blocked Clients Guide](https://redis.io/topics/clients)
