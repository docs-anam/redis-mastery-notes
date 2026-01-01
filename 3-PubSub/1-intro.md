# Introduction to Redis Pub/Sub

## Overview
Redis Pub/Sub (Publish/Subscribe) is a real-time messaging system that enables decoupled communication between producers (publishers) and consumers (subscribers). It's designed for broadcasting messages to multiple recipients without requiring them to store or poll for updates.

## Key Features
- **Real-time messaging**: Immediate message delivery as they're published
- **Decoupled architecture**: Publishers and subscribers don't need to know each other
- **Pattern-based subscriptions**: Subscribe to channels using wildcard patterns
- **Fire-and-forget**: Messages aren't persisted; late subscribers miss messages
- **Multiple channels**: Subscribe to multiple channels simultaneously
- **Low latency**: Optimized for fast message delivery

## How Pub/Sub Works

```
┌─────────────────────────────────────────────────┐
│           Redis Pub/Sub Architecture            │
├─────────────────────────────────────────────────┤
│                                                  │
│  Publishers                 Redis Server         │
│  ┌──────────┐              ┌──────────────┐    │
│  │Publisher │ ─publish──>  │   Channels   │    │
│  └──────────┘              │  • chat      │    │
│  ┌──────────┐              │  • orders    │    │
│  │Publisher │              │  • alerts    │    │
│  └──────────┘              └──────────────┘    │
│                                    │             │
│                            subscribe            │
│                                    │             │
│  Subscribers                        ↓           │
│  ┌──────────┐              ┌──────────────┐    │
│  │Subscriber│              │  Subscribers │    │
│  └──────────┘              │  Queue       │    │
│  ┌──────────┐              └──────────────┘    │
│  │Subscriber│                                  │
│  └──────────┘                                  │
│                                                  │
└─────────────────────────────────────────────────┘
```

## Basic Workflow

### 1. Publisher Sends Message
```redis
PUBLISH orders "order:12345:completed"
// Returns: number of subscribers that received the message
```

### 2. Subscribers Receive Message
Subscribers listening to "orders" channel receive the message instantly.

```redis
SUBSCRIBE orders
// Receives: message from orders channel
```

## Core Concepts

### Channels
- **Named topics** for organizing messages
- No predefined schema required
- Created dynamically on first subscription
- Case-sensitive

### Subscribers
- Clients listening to one or more channels
- Receive messages published to those channels
- Can use pattern subscriptions

### Publishers
- Send messages to channels
- Don't need to know who's listening
- Publish and forget (no persistence)

## Message Format

When a subscriber receives a message, it gets three pieces of information:

```
1. Message type: "message"
2. Channel name: "orders"
3. Message content: "order:12345:completed"
```

## Pub/Sub vs Other Solutions

| Feature | Pub/Sub | Lists (Queue) | Streams | Persistence |
|---------|---------|---------------|---------|-------------|
| Real-time | ✅ | ❌ | ✅ | ❌ |
| Persistence | ❌ | ✅ | ✅ | ✅ |
| Multiple subscribers | ✅ | ❌ (blocking) | ✅ | ✅ |
| Late subscribers | ❌ | ✅ | ✅ | ✅ |
| Pattern matching | ✅ | ❌ | ❌ | ❌ |
| Latency | Very low | Low | Low | Low |

## Use Cases

### 1. Real-time Notifications
Broadcast notifications to connected users.

### 2. Chat Systems
Send messages between users in real-time.

### 3. Live Updates
Push updates to dashboards and web clients.

### 4. Event Broadcasting
Notify multiple systems of events (order placed, user signed up).

### 5. IoT Data Distribution
Stream sensor data to multiple consumers.

### 6. Cache Invalidation
Notify all servers when cache needs refresh.

## Quick Start Example

### Publisher (Producer)
```python
import redis

r = redis.Redis()
r.publish('notifications', 'New order placed!')
r.publish('notifications', 'User signed up!')
```

### Subscriber (Consumer)
```python
import redis

r = redis.Redis()
pubsub = r.pubsub()
pubsub.subscribe('notifications')

for message in pubsub.listen():
    if message['type'] == 'message':
        print(f"Received: {message['data']}")
```

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| PUBLISH | O(N+M) | N = channels, M = patterns |
| SUBSCRIBE | O(1) | Instant subscription |
| Message delivery | O(1) | Per subscriber |
| Pattern matching | O(N) | N = subscribed patterns |

## Memory Considerations

- **No message storage**: Messages aren't stored
- **Subscription metadata**: ~200 bytes per active subscription
- **No backlog**: Memory freed immediately after delivery
- **Scalable**: Millions of subscribers possible

## Common Patterns

### Pattern Subscription
Subscribe to channel families with wildcards.

```
news:* → Matches: news:sports, news:tech, news:health
user:*:notifications → Matches: user:123:notifications, user:456:notifications
```

### Channel Naming Convention
Use hierarchical naming for organization.

```
user:123:notifications
order:456:status
payment:invoice:789
alerts:critical:cpu
```

## When to Use Pub/Sub

✅ **Good for:**
- Real-time messaging
- Broadcasting to multiple clients
- Live feeds and notifications
- Cache invalidation
- Event-driven architectures

❌ **NOT good for:**
- Message persistence needed
- Late subscribers need old messages
- Guaranteed delivery required
- Complex routing logic
- Message acknowledgment needed

## When NOT to Use Pub/Sub

If you need:
- **Message persistence**: Use Streams or Lists
- **Message reliability**: Use Streams with consumer groups
- **Complex filtering**: Use Streams with consumer groups
- **Message history**: Use Streams

## Architecture Patterns

### Simple Publish-Subscribe
One publisher, multiple subscribers on same channel.

### Hierarchical Topics
Publishers send to different channels based on type.

### Fan-out Broadcasting
One event triggers messages to multiple channels.

### Request-Reply
Use two channels (request, response) for RPC-like patterns.

## Best Practices

1. **Use meaningful channel names**: Hierarchical and descriptive
2. **Handle connection losses**: Subscribers need reconnection logic
3. **Know your limits**: Pub/Sub doesn't scale to all use cases
4. **Monitor subscribers**: Track active subscriptions
5. **Use pattern subscriptions carefully**: Can be resource intensive
6. **Plan for fallback**: Have alternative communication method ready
7. **Document channels**: Keep list of active channels and purposes

## Common Pitfalls

1. **Expecting persistence**: Messages disappear if no subscribers
2. **Late subscribers missing messages**: Fire-and-forget model
3. **No delivery guarantees**: Messages can be lost
4. **Assuming message ordering**: Not guaranteed across multiple publishers
5. **Too many patterns**: Pattern matching can be slow
6. **Connection issues**: Network drops lose subscription

## Next Steps

- [Channels](2-channel.md) - Understanding channel organization and patterns
- [Subscribers](3-subscriber.md) - Implementation and best practices
- [Publishers](4-publisher.md) - Sending messages effectively
- [Limitations](5-limitation.md) - Understanding constraints and alternatives
