# Redis Pub/Sub Publishers

## Overview
Publishers send messages to channels. Unlike subscribers which maintain persistent connections, publishers can publish and disconnect without affecting message delivery. This document covers publishing strategies, optimization techniques, and common patterns.

## Publisher Basics

### Publishing Model
- **Fire-and-forget**: Publish and don't wait for confirmation
- **Returns subscriber count**: Tells you how many subscribers got the message
- **No persistence**: Message is lost if no one is listening
- **Instant delivery**: Messages delivered immediately to connected subscribers
- **Stateless**: Publishers don't maintain connection state

### PUBLISH Command

```redis
PUBLISH channel message
// Returns: number of subscribers that received the message (0 if none listening)
```

Basic example:
```redis
PUBLISH orders "order:12345:placed"
// Returns: 3 (if 3 subscribers were listening)

PUBLISH orders "order:12345:placed"
// Returns: 0 (if no subscribers were listening)
```

## Basic Publisher Implementation

### Python Publisher

#### Simple Publishing
```python
import redis

r = redis.Redis(host='localhost', port=6379)

# Single message
num_subscribers = r.publish('orders', 'order:12345:placed')
print(f"Message delivered to {num_subscribers} subscribers")

# Multiple messages to same channel
r.publish('orders', 'order:12346:placed')
r.publish('orders', 'order:12347:shipped')

# Multiple channels
r.publish('orders', 'order placed')
r.publish('notifications', 'new notification')
```

#### Publisher with Verification
```python
import redis

class Publisher:
    def __init__(self, host='localhost', port=6379):
        self.r = redis.Redis(host=host, port=port)
    
    def publish(self, channel, message):
        """Publish message and return subscriber count"""
        try:
            num_subscribers = self.r.publish(channel, message)
            return num_subscribers
        except redis.ConnectionError as e:
            print(f"Error publishing to {channel}: {e}")
            return -1
    
    def publish_with_check(self, channel, message, min_subscribers=1):
        """Publish only if minimum subscribers present"""
        num_subscribers = self.publish(channel, message)
        if num_subscribers < min_subscribers:
            print(f"Warning: Only {num_subscribers} subscribers for {channel}")
        return num_subscribers

# Usage
publisher = Publisher()
num = publisher.publish('orders', 'order:123:placed')
print(f"Published to {num} subscribers")
```

### Node.js Publisher

```javascript
const redis = require('redis');

class Publisher {
    constructor() {
        this.client = redis.createClient();
    }
    
    publish(channel, message) {
        return new Promise((resolve, reject) => {
            this.client.publish(channel, message, (err, numSubscribers) => {
                if (err) {
                    reject(err);
                } else {
                    resolve(numSubscribers);
                }
            });
        });
    }
    
    async publishJSON(channel, data) {
        try {
            const message = JSON.stringify(data);
            const numSubscribers = await this.publish(channel, message);
            console.log(`Published to ${numSubscribers} subscribers`);
            return numSubscribers;
        } catch (err) {
            console.error('Publish error:', err);
            return -1;
        }
    }
}

// Usage
const publisher = new Publisher();
publisher.publishJSON('orders', {
    orderId: 123,
    status: 'placed',
    total: 99.99
});
```

## Publishing Patterns

### Pattern 1: Event-Based Publishing

```python
import json
import time

class EventPublisher:
    def __init__(self):
        self.r = redis.Redis()
    
    def publish_event(self, event_type, entity_id, data):
        """Publish typed events"""
        channel = f"{event_type}:{entity_id}"
        message = {
            'type': event_type,
            'entity_id': entity_id,
            'timestamp': time.time(),
            'data': data
        }
        self.r.publish(channel, json.dumps(message))
    
    def publish_order_event(self, order_id, status, details=None):
        """Publish order events"""
        self.publish_event('order', order_id, {
            'status': status,
            'details': details
        })

# Usage
publisher = EventPublisher()
publisher.publish_order_event(12345, 'placed', {'total': 99.99})
publisher.publish_order_event(12346, 'shipped', {'carrier': 'FedEx'})
```

### Pattern 2: Hierarchical Channel Publishing

```python
class HierarchicalPublisher:
    def __init__(self):
        self.r = redis.Redis()
    
    def publish_to_hierarchy(self, category, action, data):
        """Publish to multiple levels of hierarchy"""
        message = json.dumps(data)
        
        # Publish to specific channel
        self.r.publish(f"{category}:{action}", message)
        
        # Publish to broader category
        self.r.publish(f"{category}:all", message)
        
        # Publish to all-events channel
        self.r.publish("events:all", message)
    
    def publish_order_update(self, order_id, status):
        self.publish_to_hierarchy('orders', status, {
            'order_id': order_id,
            'status': status,
            'timestamp': time.time()
        })

# Usage
publisher = HierarchicalPublisher()
publisher.publish_order_update(12345, 'placed')
# Publishes to: orders:placed, orders:all, events:all
```

### Pattern 3: Batch Publishing

```python
import json
from typing import List, Dict

class BatchPublisher:
    def __init__(self):
        self.r = redis.Redis()
    
    def publish_batch(self, channel, messages: List[Dict]):
        """Publish multiple messages efficiently"""
        count = 0
        for message in messages:
            num_subscribers = self.r.publish(
                channel, 
                json.dumps(message)
            )
            if num_subscribers > 0:
                count += 1
        return count
    
    def publish_in_batches(self, channel, messages: List[Dict], batch_size=100):
        """Publish large number of messages in batches"""
        for i in range(0, len(messages), batch_size):
            batch = messages[i:i + batch_size]
            count = self.publish_batch(channel, batch)
            print(f"Batch {i//batch_size}: Published to {count} subscribers")

# Usage
publisher = BatchPublisher()
messages = [
    {'order_id': i, 'amount': 100 + i}
    for i in range(1000)
]
publisher.publish_in_batches('orders', messages)
```

### Pattern 4: Conditional Publishing

```python
class ConditionalPublisher:
    def __init__(self):
        self.r = redis.Redis()
    
    def publish_if_subscribers(self, channel, message, min_subs=1):
        """Only publish if subscribers present"""
        # First check if channel has subscribers
        # Note: This is approximate, not exact
        num_subs = self.r.publish(channel, message)
        return num_subs >= min_subs
    
    def publish_with_fallback(self, primary_channel, fallback_channel, message):
        """Try primary channel, fall back if no subscribers"""
        num_subs = self.r.publish(primary_channel, message)
        
        if num_subs == 0:
            print(f"No subscribers on {primary_channel}, using {fallback_channel}")
            num_subs = self.r.publish(fallback_channel, message)
        
        return num_subs
    
    def publish_to_multiple(self, channels, message):
        """Publish to multiple channels"""
        total = 0
        for channel in channels:
            total += self.r.publish(channel, message)
        return total

# Usage
publisher = ConditionalPublisher()
publisher.publish_if_subscribers('orders', 'message', min_subs=1)
publisher.publish_with_fallback('priority', 'default', 'message')
```

## Message Formatting

### JSON Messages

```python
import json

class JSONPublisher:
    def __init__(self):
        self.r = redis.Redis()
    
    def publish_json(self, channel, data):
        """Publish JSON data"""
        message = json.dumps(data)
        return self.r.publish(channel, message)
    
    def publish_order(self, order_id, status, details):
        return self.publish_json('orders', {
            'order_id': order_id,
            'status': status,
            'details': details,
            'timestamp': time.time()
        })

# Usage
publisher = JSONPublisher()
publisher.publish_order(12345, 'placed', {
    'user_id': 789,
    'total': 99.99,
    'items': 3
})
```

### Compressed Messages

```python
import json
import gzip
import base64

class CompressedPublisher:
    def __init__(self):
        self.r = redis.Redis()
    
    def publish_compressed(self, channel, data):
        """Publish compressed JSON"""
        json_str = json.dumps(data)
        compressed = gzip.compress(json_str.encode())
        encoded = base64.b64encode(compressed).decode()
        
        return self.r.publish(channel, f"ZLIB:{encoded}")
    
    def publish_large_data(self, channel, data):
        """Compress large data before publishing"""
        json_size = len(json.dumps(data).encode())
        
        if json_size > 1000:  # Compress if > 1KB
            return self.publish_compressed(channel, data)
        else:
            return self.r.publish(channel, json.dumps(data))

# Usage
publisher = CompressedPublisher()
large_data = {'items': list(range(1000))}
publisher.publish_large_data('data', large_data)
```

### Structured Messages

```python
import json
from dataclasses import dataclass
from typing import Any

@dataclass
class Message:
    event_type: str
    entity_id: str
    timestamp: float
    source: str
    data: Any
    
    def to_json(self):
        return json.dumps({
            'event_type': self.event_type,
            'entity_id': self.entity_id,
            'timestamp': self.timestamp,
            'source': self.source,
            'data': self.data
        })

class StructuredPublisher:
    def __init__(self, source='app'):
        self.r = redis.Redis()
        self.source = source
    
    def publish_message(self, channel, message: Message):
        return self.r.publish(channel, message.to_json())
    
    def publish_order_event(self, order_id, event_type, data):
        message = Message(
            event_type=event_type,
            entity_id=str(order_id),
            timestamp=time.time(),
            source=self.source,
            data=data
        )
        return self.publish_message(f"orders:{event_type}", message)

# Usage
publisher = StructuredPublisher(source='order-service')
publisher.publish_order_event(12345, 'placed', {'total': 99.99})
```

## Performance Optimization

### Connection Pooling

```python
import redis

class PooledPublisher:
    def __init__(self, host='localhost', port=6379, max_connections=10):
        self.pool = redis.ConnectionPool(
            host=host,
            port=port,
            max_connections=max_connections,
            decode_responses=True
        )
        self.r = redis.Redis(connection_pool=self.pool)
    
    def publish(self, channel, message):
        return self.r.publish(channel, message)
    
    def close(self):
        self.pool.disconnect()

# Usage
publisher = PooledPublisher()
for i in range(1000):
    publisher.publish('channel', f'message:{i}')
publisher.close()
```

### Pipelined Publishing

```python
class PipelinePublisher:
    def __init__(self):
        self.r = redis.Redis()
    
    def publish_multiple(self, messages):
        """Publish multiple messages using pipeline"""
        pipe = self.r.pipeline()
        
        for channel, message in messages:
            pipe.publish(channel, message)
        
        results = pipe.execute()
        return sum(results)  # Total subscribers
    
    def batch_publish(self, channel, messages, batch_size=100):
        """Publish messages in batches using pipelines"""
        total_subs = 0
        
        for i in range(0, len(messages), batch_size):
            batch = messages[i:i + batch_size]
            
            pipe = self.r.pipeline()
            for message in batch:
                pipe.publish(channel, message)
            
            results = pipe.execute()
            total_subs += sum(results)
        
        return total_subs

# Usage
publisher = PipelinePublisher()

# Method 1: Multi-channel batch
messages = [
    ('orders', 'msg1'),
    ('orders', 'msg2'),
    ('notifications', 'msg3'),
]
publisher.publish_multiple(messages)

# Method 2: Single channel batches
messages = [f'message:{i}' for i in range(1000)]
publisher.batch_publish('channel', messages)
```

## Error Handling and Reliability

### Retry Publishing

```python
import redis
import time
from functools import wraps

def retry_publish(max_retries=3, backoff=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except redis.ConnectionError:
                    if attempt < max_retries - 1:
                        wait = backoff ** attempt
                        print(f"Retry in {wait}s...")
                        time.sleep(wait)
                    else:
                        raise
        return wrapper
    return decorator

class ResilientPublisher:
    def __init__(self):
        self.r = redis.Redis()
    
    @retry_publish(max_retries=3, backoff=2)
    def publish(self, channel, message):
        return self.r.publish(channel, message)

# Usage
publisher = ResilientPublisher()
try:
    publisher.publish('orders', 'message')
except redis.ConnectionError:
    print("Failed after retries")
```

### Circuit Breaker Pattern

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = 1
    OPEN = 2
    HALF_OPEN = 3

class CircuitBreakerPublisher:
    def __init__(self, failure_threshold=5, timeout=60):
        self.r = redis.Redis()
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
    
    def publish(self, channel, message):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = self.r.publish(channel, message)
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
            return result
        except redis.ConnectionError:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
            raise

# Usage
publisher = CircuitBreakerPublisher()
try:
    publisher.publish('channel', 'message')
except Exception as e:
    print(f"Circuit breaker: {e}")
```

## Monitoring and Logging

### Publishing Analytics

```python
import redis
import json
from datetime import datetime

class AnalyticsPublisher:
    def __init__(self):
        self.r = redis.Redis()
        self.stats = {
            'total_published': 0,
            'total_subscribers': 0,
            'channels': {}
        }
    
    def publish(self, channel, message):
        num_subs = self.r.publish(channel, message)
        
        # Track statistics
        self.stats['total_published'] += 1
        self.stats['total_subscribers'] += num_subs
        
        if channel not in self.stats['channels']:
            self.stats['channels'][channel] = {
                'count': 0,
                'last_pub': None,
                'subscribers': 0
            }
        
        self.stats['channels'][channel]['count'] += 1
        self.stats['channels'][channel]['last_pub'] = datetime.now().isoformat()
        self.stats['channels'][channel]['subscribers'] = num_subs
        
        return num_subs
    
    def get_stats(self):
        return self.stats

# Usage
publisher = AnalyticsPublisher()
publisher.publish('orders', 'msg1')
publisher.publish('orders', 'msg2')
publisher.publish('notifications', 'msg3')
print(json.dumps(publisher.get_stats(), indent=2))
```

### Request Logging

```python
import logging
import json

class LoggedPublisher:
    def __init__(self):
        self.r = redis.Redis()
        self.logger = logging.getLogger(__name__)
    
    def publish(self, channel, message):
        try:
            num_subs = self.r.publish(channel, message)
            self.logger.info(
                f"Published to {channel}",
                extra={
                    'channel': channel,
                    'subscribers': num_subs,
                    'message_size': len(str(message))
                }
            )
            return num_subs
        except Exception as e:
            self.logger.error(
                f"Publish failed to {channel}",
                exc_info=True,
                extra={'channel': channel}
            )
            raise

# Usage
logging.basicConfig(level=logging.INFO)
publisher = LoggedPublisher()
publisher.publish('orders', json.dumps({'order_id': 123}))
```

## Best Practices

### 1. Use Structured Data
```python
# Good
message = json.dumps({'order_id': 123, 'status': 'placed'})
r.publish('orders', message)

# Bad
r.publish('orders', 'order 123 placed')
```

### 2. Include Timestamps
```python
# Good
message = {
    'order_id': 123,
    'timestamp': time.time(),
    'status': 'placed'
}
r.publish('orders', json.dumps(message))
```

### 3. Use Appropriate Channel Names
```python
# Good
r.publish('orders:placed', message)
r.publish('orders:shipped', message)

# Bad
r.publish('o', message)
r.publish('x', message)
```

### 4. Handle Publishing Failures
```python
try:
    num_subs = r.publish('orders', message)
    if num_subs == 0:
        logger.warning(f"No subscribers for orders channel")
except redis.ConnectionError:
    logger.error("Redis connection failed")
    # Implement fallback
```

### 5. Monitor Subscriber Counts
```python
# Track if messages are reaching subscribers
for i in range(100):
    num = r.publish('orders', f'message:{i}')
    if num == 0:
        print(f"Message {i}: No subscribers")
    else:
        print(f"Message {i}: {num} subscribers")
```

## Common Patterns Summary

| Pattern | Use Case | Example |
|---------|----------|---------|
| Event-based | Domain events | order:placed, user:signup |
| Hierarchical | Multi-level routing | orders:status, orders:all |
| Batch | High-volume publishing | Publishing 1000 messages |
| Conditional | Subscriber-aware | Only publish if listeners |
| JSON | Structured data | Objects with metadata |

## Performance Characteristics

| Aspect | Performance | Notes |
|--------|-----------|-------|
| Publishing speed | Sub-millisecond | Single message |
| Throughput | 100k+ msg/sec | Depends on hardware |
| Latency | < 1ms | To connected subscribers |
| Memory | None | No message storage |
| Scalability | Very high | Limited by network |

## Summary

Publishers are simple but powerful:
- Use structured message formats (JSON)
- Include metadata (timestamps, IDs)
- Implement error handling and retries
- Monitor publisher health and stats
- Design channels hierarchically
- Consider batch publishing for high throughput
