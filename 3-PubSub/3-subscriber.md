# Redis Pub/Sub Subscribers

## Overview
Subscribers are clients that listen to one or more channels for incoming messages. Once subscribed, a client enters a special mode where it can only perform subscription-related commands and message listening. This document covers implementation patterns, best practices, and common challenges.

## Subscriber Basics

### Subscription Modes
Once a client subscribes to any channel, it enters **subscription mode** where:
- Can only use subscription commands: SUBSCRIBE, PSUBSCRIBE, UNSUBSCRIBE, PUNSUBSCRIBE, QUIT
- Cannot use regular commands: SET, GET, DEL, etc.
- Must exit subscription mode to use other commands
- Used dedicated connection for subscriptions

### Subscription Commands

| Command | Purpose | Syntax | Returns |
|---------|---------|--------|---------|
| SUBSCRIBE | Subscribe to channels | SUBSCRIBE channel [channel ...] | Subscription confirmation |
| PSUBSCRIBE | Pattern subscription | PSUBSCRIBE pattern [pattern ...] | Pattern subscription confirmation |
| UNSUBSCRIBE | Unsubscribe from channels | UNSUBSCRIBE [channel ...] | Unsubscribe confirmation |
| PUNSUBSCRIBE | Pattern unsubscribe | PUNSUBSCRIBE [pattern ...] | Pattern unsubscribe confirmation |
| QUIT | Exit subscription mode | QUIT | OK |

### Message Format

Subscribers receive messages in a standardized format:

```
Type 1: Subscribe Confirmation
['subscribe', 'channel_name', subscriber_count]

Type 2: Message Received
['message', 'channel_name', 'message_content']

Type 3: Unsubscribe Confirmation
['unsubscribe', 'channel_name', subscriber_count]

Type 4: Pattern Subscription Confirmation
['psubscribe', 'pattern', subscriber_count]

Type 5: Pattern Message
['pmessage', 'pattern', 'channel_name', 'message_content']
```

## Basic Subscriber Implementation

### Simple Subscriber (Redis CLI)
```redis
SUBSCRIBE orders notifications
// Once connected, receives all messages from both channels
```

### Python Subscriber

#### Basic Implementation
```python
import redis

r = redis.Redis(host='localhost', port=6379)
pubsub = r.pubsub()

# Subscribe to channels
pubsub.subscribe('orders', 'notifications')

# Listen for messages
for message in pubsub.listen():
    if message['type'] == 'message':
        channel = message['channel']
        data = message['data']
        print(f"[{channel}] {data}")
```

#### With Error Handling
```python
import redis
import logging

logger = logging.getLogger(__name__)

class Subscriber:
    def __init__(self, channels):
        self.r = redis.Redis()
        self.pubsub = self.r.pubsub()
        self.channels = channels
    
    def subscribe(self):
        try:
            self.pubsub.subscribe(*self.channels)
            logger.info(f"Subscribed to: {self.channels}")
        except Exception as e:
            logger.error(f"Subscription error: {e}")
    
    def listen(self):
        try:
            for message in self.pubsub.listen():
                self.handle_message(message)
        except redis.ConnectionError:
            logger.error("Connection lost, attempting reconnect...")
            self.reconnect()
    
    def handle_message(self, message):
        if message['type'] == 'message':
            self.process(message['channel'], message['data'])
        elif message['type'] == 'pmessage':
            self.process_pattern(message)
    
    def process(self, channel, data):
        print(f"Message on {channel}: {data}")
    
    def process_pattern(self, message):
        print(f"Pattern match: {message['pattern']} -> {message['channel']}: {message['data']}")
    
    def reconnect(self):
        self.pubsub.close()
        self.subscribe()

# Usage
subscriber = Subscriber(['orders', 'notifications'])
subscriber.subscribe()
subscriber.listen()
```

### Node.js Subscriber

```javascript
const redis = require('redis');

class Subscriber {
    constructor(channels) {
        this.client = redis.createClient();
        this.channels = channels;
    }
    
    subscribe() {
        this.client.on('connect', () => {
            console.log('Connected to Redis');
            this.client.subscribe(this.channels, (err, count) => {
                if (err) {
                    console.error('Subscribe error:', err);
                } else {
                    console.log(`Subscribed to ${count} channels`);
                }
            });
        });
        
        this.client.on('message', (channel, message) => {
            this.handleMessage(channel, message);
        });
        
        this.client.on('error', (err) => {
            console.error('Redis error:', err);
        });
    }
    
    handleMessage(channel, message) {
        console.log(`[${channel}] ${message}`);
    }
}

// Usage
const subscriber = new Subscriber(['orders', 'notifications']);
subscriber.subscribe();
```

## Pattern Subscription

### Pattern-based Listening

```python
import redis

pubsub = redis.Redis().pubsub()

# Subscribe to all order-related channels
pubsub.psubscribe('orders:*')

# Subscribe to user notifications
pubsub.psubscribe('user:*:notifications')

# Listen for messages
for message in pubsub.listen():
    if message['type'] == 'pmessage':
        pattern = message['pattern']
        channel = message['channel']
        data = message['data']
        print(f"[{pattern}] -> [{channel}] {data}")
```

### Pattern Matching Examples

```python
pubsub = redis.Redis().pubsub()

# Match all error channels
pubsub.psubscribe('*:error')

# Match status updates for specific service
pubsub.psubscribe('service:inventory:*')

# Match all user-specific messages
pubsub.psubscribe('user:*:*')

# Match multi-level patterns
pubsub.psubscribe('orders:*:*:notification')
```

## Subscriber Patterns

### Pattern 1: Single-Purpose Subscriber
Dedicated subscriber for specific channels.

```python
class OrderSubscriber:
    def __init__(self):
        self.pubsub = redis.Redis().pubsub()
    
    def start(self):
        self.pubsub.subscribe('orders:placed', 'orders:shipped')
        
        for message in self.pubsub.listen():
            if message['type'] == 'message':
                self.process_order_event(message)
    
    def process_order_event(self, message):
        # Process order-specific logic
        pass

class NotificationSubscriber:
    def __init__(self):
        self.pubsub = redis.Redis().pubsub()
    
    def start(self):
        self.pubsub.subscribe('notifications:email', 'notifications:sms')
        
        for message in self.pubsub.listen():
            if message['type'] == 'message':
                self.process_notification(message)
```

### Pattern 2: Multi-Channel Router
Route messages to different handlers.

```python
class MessageRouter:
    def __init__(self):
        self.pubsub = redis.Redis().pubsub()
        self.handlers = {
            'orders': self.handle_order,
            'payments': self.handle_payment,
            'notifications': self.handle_notification
        }
    
    def subscribe(self):
        channels = list(self.handlers.keys())
        self.pubsub.subscribe(channels)
    
    def listen(self):
        for message in self.pubsub.listen():
            if message['type'] == 'message':
                channel = message['channel']
                for key, handler in self.handlers.items():
                    if channel.startswith(key):
                        handler(message['data'])
    
    def handle_order(self, data):
        print(f"Order handler: {data}")
    
    def handle_payment(self, data):
        print(f"Payment handler: {data}")
    
    def handle_notification(self, data):
        print(f"Notification handler: {data}")
```

### Pattern 3: Pattern-based Subscription
Dynamic subscriptions based on patterns.

```python
class PatternSubscriber:
    def __init__(self):
        self.pubsub = redis.Redis().pubsub()
        self.patterns = []
    
    def subscribe_to_pattern(self, pattern):
        self.pubsub.psubscribe(pattern)
        self.patterns.append(pattern)
        print(f"Subscribed to pattern: {pattern}")
    
    def listen(self):
        for message in self.pubsub.listen():
            if message['type'] == 'pmessage':
                self.process_pattern_message(message)
    
    def process_pattern_message(self, message):
        pattern = message['pattern']
        channel = message['channel']
        data = message['data']
        print(f"Pattern '{pattern}' matched on '{channel}': {data}")
    
    def unsubscribe_from_pattern(self, pattern):
        self.pubsub.punsubscribe(pattern)
        self.patterns.remove(pattern)

# Usage
subscriber = PatternSubscriber()
subscriber.subscribe_to_pattern('user:*:notifications')
subscriber.subscribe_to_pattern('order:*:updates')
subscriber.listen()
```

### Pattern 4: Threaded Subscriber
Background listener with separate processing.

```python
import threading
import redis
from queue import Queue

class ThreadedSubscriber:
    def __init__(self, channels):
        self.message_queue = Queue()
        self.pubsub = redis.Redis().pubsub()
        self.channels = channels
        self.running = False
    
    def start(self):
        self.running = True
        
        # Listener thread
        listener_thread = threading.Thread(
            target=self.listen_loop,
            daemon=True
        )
        listener_thread.start()
        
        # Processor thread
        processor_thread = threading.Thread(
            target=self.process_loop,
            daemon=True
        )
        processor_thread.start()
    
    def listen_loop(self):
        self.pubsub.subscribe(*self.channels)
        for message in self.pubsub.listen():
            if not self.running:
                break
            if message['type'] == 'message':
                self.message_queue.put(message)
    
    def process_loop(self):
        while self.running:
            try:
                message = self.message_queue.get(timeout=1)
                self.process_message(message)
            except:
                pass
    
    def process_message(self, message):
        print(f"Processing: {message}")
    
    def stop(self):
        self.running = False
        self.pubsub.close()

# Usage
subscriber = ThreadedSubscriber(['orders', 'notifications'])
subscriber.start()
```

## Connection Management

### Handling Disconnections

```python
import redis
import time

class ResilientSubscriber:
    def __init__(self, channels, max_retries=5):
        self.channels = channels
        self.max_retries = max_retries
        self.retry_count = 0
    
    def connect(self):
        try:
            r = redis.Redis(
                host='localhost',
                port=6379,
                socket_connect_timeout=5,
                socket_keepalive=True
            )
            r.ping()
            return r
        except redis.ConnectionError as e:
            print(f"Connection error: {e}")
            return None
    
    def start(self):
        while self.retry_count < self.max_retries:
            try:
                r = self.connect()
                if not r:
                    self.retry_count += 1
                    time.sleep(2 ** self.retry_count)
                    continue
                
                pubsub = r.pubsub()
                pubsub.subscribe(*self.channels)
                self.retry_count = 0  # Reset on success
                
                for message in pubsub.listen():
                    self.handle_message(message)
                    
            except redis.ConnectionError:
                self.retry_count += 1
                print(f"Reconnecting... (attempt {self.retry_count})")
                time.sleep(2 ** self.retry_count)
        
        print("Max retries reached")
    
    def handle_message(self, message):
        if message['type'] == 'message':
            print(f"Message: {message}")
```

### Connection Pooling

```python
import redis

class PooledSubscriber:
    def __init__(self, channels):
        self.pool = redis.ConnectionPool(
            host='localhost',
            port=6379,
            max_connections=5,
            decode_responses=True
        )
        self.channels = channels
    
    def subscribe(self):
        r = redis.Redis(connection_pool=self.pool)
        pubsub = r.pubsub(ignore_subscribe_messages=True)
        pubsub.subscribe(*self.channels)
        
        for message in pubsub.listen():
            self.handle_message(message)
    
    def handle_message(self, message):
        print(f"Received: {message}")
```

## Message Processing

### JSON Message Handling

```python
import json
import redis

class JSONSubscriber:
    def __init__(self, channels):
        self.pubsub = redis.Redis().pubsub()
        self.channels = channels
    
    def start(self):
        self.pubsub.subscribe(*self.channels)
        
        for message in self.pubsub.listen():
            if message['type'] == 'message':
                try:
                    data = json.loads(message['data'])
                    self.process_json(message['channel'], data)
                except json.JSONDecodeError:
                    print(f"Invalid JSON: {message['data']}")
    
    def process_json(self, channel, data):
        print(f"[{channel}] Parsed JSON: {data}")

# Usage
subscriber = JSONSubscriber(['orders', 'events'])
subscriber.start()
```

### Filtering and Processing

```python
import redis
import json

class FilteringSubscriber:
    def __init__(self):
        self.pubsub = redis.Redis().pubsub()
    
    def start(self):
        self.pubsub.psubscribe('order:*')
        
        for message in self.pubsub.listen():
            if message['type'] == 'pmessage':
                data = json.loads(message['data'])
                
                # Filter by status
                if data.get('status') == 'critical':
                    self.handle_critical(data)
                elif data.get('status') == 'warning':
                    self.handle_warning(data)
    
    def handle_critical(self, data):
        print(f"CRITICAL: {data}")
    
    def handle_warning(self, data):
        print(f"WARNING: {data}")
```

## Performance Optimization

### Batch Processing

```python
import redis
from collections import deque
import time

class BatchSubscriber:
    def __init__(self, channels, batch_size=100, timeout=5):
        self.pubsub = redis.Redis().pubsub()
        self.channels = channels
        self.batch_size = batch_size
        self.timeout = timeout
        self.batch = deque()
        self.last_flush = time.time()
    
    def start(self):
        self.pubsub.subscribe(*self.channels)
        
        for message in self.pubsub.listen():
            if message['type'] == 'message':
                self.batch.append(message)
                
                # Flush on size or timeout
                if (len(self.batch) >= self.batch_size or 
                    time.time() - self.last_flush > self.timeout):
                    self.process_batch()
    
    def process_batch(self):
        if self.batch:
            print(f"Processing batch of {len(self.batch)} messages")
            # Process all at once
            self.batch.clear()
            self.last_flush = time.time()
```

### Priority Queue Handling

```python
import redis
import json
from heapq import heappush, heappop
import threading

class PrioritySubscriber:
    def __init__(self):
        self.pubsub = redis.Redis().pubsub()
        self.priority_queue = []
        self.lock = threading.Lock()
    
    def start(self):
        self.pubsub.psubscribe('*:priority')
        
        for message in self.pubsub.listen():
            if message['type'] == 'pmessage':
                try:
                    data = json.loads(message['data'])
                    priority = data.get('priority', 5)
                    
                    with self.lock:
                        heappush(self.priority_queue, 
                                (priority, message))
                except:
                    pass
        
        self.process_priority_queue()
    
    def process_priority_queue(self):
        while self.priority_queue:
            with self.lock:
                priority, message = heappop(self.priority_queue)
            print(f"Processing priority {priority}: {message}")
```

## Best Practices

### 1. Use Dedicated Connections
```python
# Good: Separate connection for subscriptions
r = redis.Redis()  # Regular commands
pubsub = redis.Redis().pubsub()  # Subscriptions
```

### 2. Handle All Message Types
```python
for message in pubsub.listen():
    if message['type'] == 'subscribe':
        print(f"Subscribed to {message['channel']}")
    elif message['type'] == 'message':
        print(f"Message: {message['data']}")
    elif message['type'] == 'unsubscribe':
        print(f"Unsubscribed from {message['channel']}")
```

### 3. Implement Reconnection Logic
```python
# Exponential backoff for reconnection
retry_delay = 1
while True:
    try:
        subscriber.start()
    except redis.ConnectionError:
        print(f"Reconnecting in {retry_delay}s")
        time.sleep(retry_delay)
        retry_delay = min(retry_delay * 2, 60)
```

### 4. Monitor Subscription Health
```python
class HealthCheckSubscriber:
    def __init__(self):
        self.last_message_time = time.time()
        self.timeout = 30
    
    def check_health(self):
        if time.time() - self.last_message_time > self.timeout:
            print("No messages received - checking connection")
            self.reconnect()
    
    def handle_message(self, message):
        self.last_message_time = time.time()
        # Process message
```

## Common Issues and Solutions

### Issue: Blocking Forever on Listen
**Solution**: Add timeout handling
```python
try:
    for message in pubsub.listen(ignore_subscribe_messages=True):
        handle(message)
except redis.ConnectionError:
    print("Connection lost")
```

### Issue: Memory Leak from Accumulating Subscriptions
**Solution**: Unsubscribe when done
```python
pubsub.unsubscribe('old_channel')
pubsub.punsubscribe('old_pattern')
```

### Issue: Late Message Loss
**Solution**: Use Streams instead for message history
```python
# Pub/Sub doesn't have history
# Use Streams (XREAD) for guaranteed message delivery
```

### Issue: Message Order Not Guaranteed
**Solution**: Add sequence numbers
```python
message = {
    'sequence': counter,
    'timestamp': time.time(),
    'data': actual_data
}
```

## Summary

Key points for subscribers:
- Use dedicated Redis connections for subscriptions
- Handle all message types (subscribe, message, unsubscribe)
- Implement reconnection logic with exponential backoff
- Process messages based on their type and content
- Monitor connection health and handle disconnections
- Use patterns for dynamic subscriptions
- Consider Streams for guaranteed delivery when needed
