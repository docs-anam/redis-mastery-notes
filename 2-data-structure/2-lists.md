# Redis Lists - Ordered Collections & Queues

## Table of Contents
1. [Overview](#overview)
2. [List Fundamentals](#list-fundamentals)
3. [Core Commands](#core-commands)
4. [Practical Examples](#practical-examples)
5. [Real-World Patterns](#real-world-patterns)
6. [Performance Optimization](#performance-optimization)
7. [Best Practices](#best-practices)
8. [Common Mistakes](#common-mistakes)

---

## Overview

Redis Lists are ordered collections of strings, implemented as doubly-linked lists. Perfect for:
- **Task Queues**: Job processing, message queues
- **Stacks**: Undo/redo functionality
- **Activity Feeds**: User timelines, notifications
- **Sliding Windows**: Recent N items, rate limiting
- **Blocking Operations**: Producer-consumer patterns

### Why Lists?

- **O(1) operations** at both ends (LPUSH, RPUSH, LPOP, RPOP)
- **O(N) access** in middle (but mostly we work at ends)
- **Atomic operations** on sequences
- **Blocking capabilities** for real-time patterns
- **Memory efficient**: ~7 bytes per element

---

## List Fundamentals

### How Lists Work

```
RPUSH mylist a b c
HEAD                        TAIL
[a] <-> [b] <-> [c]
```

Lists are doubly-linked lists with optimizations for fast end operations.

### Core Properties

- **Ordered**: Insertion order preserved
- **Duplicates allowed**: Same value can appear multiple times
- **Bi-directional**: Fast operations from both ends
- **Index-based**: Can access by position (slow in middle)

---

## Core Commands

### Push Operations

```bash
# Push to right (append)
RPUSH queue "task1" "task2"    # Returns: 2

# Push to left (prepend)
LPUSH queue "urgent"            # Returns: 3

# Push only if exists
RPUSHX queue "task3"            # Succeeds if key exists
```

### Pop Operations

```bash
# Pop from left (FIFO)
LPOP queue                      # Returns: "urgent"

# Pop from right (LIFO)
RPOP queue                      # Returns: "task2"

# Pop with count (Redis 6.2+)
LPOP queue 2                    # Returns: ["task1", ...]
```

### Blocking Operations

```bash
# Block until element available
BLPOP queue 5                   # Wait 5 seconds
BLPOP queue 0                   # Wait forever

# Block on multiple queues
BLPOP queue1 queue2 queue3 10   # Returns [queue, value]
```

### Range & Access

```bash
# Get range
LRANGE queue 0 -1               # All elements
LRANGE queue 0 2                # First 3 elements

# Get element at index
LINDEX queue 2                  # Third element

# Get length
LLEN queue                      # Number of elements
```

### Modify Operations

```bash
# Set element at index
LSET queue 2 "new_value"

# Insert before/after
LINSERT queue BEFORE "task1" "before"
LINSERT queue AFTER "task1" "after"

# Remove elements
LREM queue 2 "value"            # Remove first 2 occurrences

# Trim (keep only range)
LTRIM queue 0 10                # Keep first 11, remove rest
```

### Search

```bash
# Find position (Redis 6.0.6+)
LPOS queue "value"              # First occurrence
LPOS queue "value" COUNT 3      # First 3 occurrences
```

---

## Practical Examples

### Example 1: Task Queue

```python
import redis
r = redis.Redis()

# Producer: Add tasks
r.rpush('task:queue', 'send_email', 'process_payment')

# Worker: Consume tasks
task = r.lpop('task:queue')
if task:
    print(f"Processing: {task}")
    process(task)
```

### Example 2: Activity Feed

```python
# Add new activity (newest first)
r.lpush('feed:user:123', 'user456 liked your post')
r.lpush('feed:user:123', 'user789 commented')

# Get recent 10
activities = r.lrange('feed:user:123', 0, 9)

# Keep only last 1000
r.ltrim('feed:user:123', 0, 999)
```

### Example 3: Blocking Queue

```python
# Producer
r.rpush('orders', 'order:1', 'order:2')

# Consumer - waits for orders
while True:
    result = r.blpop('orders', timeout=60)
    if result:
        queue_name, order = result
        process_order(order)
    else:
        print("No orders received")
```

### Example 4: Undo/Redo

```python
# User makes change
r.lpush('undo:user:123', 'delete item 5')
r.lpush('undo:user:123', 'edit item 3')

# Undo
action = r.lpop('undo:user:123')  # 'edit item 3'
r.lpush('redo:user:123', action)

# Redo
action = r.lpop('redo:user:123')
r.lpush('undo:user:123', action)
```

---

## Real-World Patterns

### Pattern 1: Rate Limiting with Sliding Window

```python
def is_rate_limited(user_id, max_requests=100, window=60):
    key = f'ratelimit:{user_id}'
    now = time.time()
    
    # Remove old entries
    r.zremrangebyscore(key, 0, now - window)
    
    # Check count
    count = r.zcard(key)
    if count < max_requests:
        r.zadd(key, {str(now): now})
        return False
    return True
```

### Pattern 2: Task Processing with Retry

```python
def process_queue_with_retry():
    while True:
        # Get task
        task = r.blpop('task:queue', timeout=10)
        if not task:
            continue
        
        task_data = json.loads(task[1])
        try:
            process(task_data)
        except Exception as e:
            retries = r.hincrby(f'task:{task_data["id"]}', 'retries', 1)
            if retries < 3:
                r.rpush('task:queue', task[1])
            else:
                r.lpush('task:queue:failed', task[1])
```

### Pattern 3: Distributed Task Queue

```python
# Multiple workers
for worker_id in range(5):
    worker = Worker(worker_id)
    worker.start()

class Worker:
    def __init__(self, worker_id):
        self.worker_id = worker_id
    
    def start(self):
        while True:
            # Each worker gets next task
            result = r.blpop('shared:queue', timeout=10)
            if result:
                self.process(result[1])
```

---

## Performance Optimization

### Operations Performance

```
Operation       Time      Memory  Notes
─────────────────────────────────────
LPUSH/RPUSH     O(1)      7 bytes per element
LPOP/RPOP       O(1)      Constant
LLEN            O(1)      Minimal
LINDEX (end)    O(1)      Constant
LINDEX (middle) O(N)      Linear scan
LRANGE          O(N)      Returns N elements
LTRIM           O(N)      Deletes elements
```

### Optimization Tips

#### Use Blocking Operations Instead of Polling

```python
# ❌ BAD: Polling with sleep
while True:
    task = r.lpop('queue')
    if task:
        process(task)
    else:
        time.sleep(0.1)  # Wastes CPU

# ✅ GOOD: Blocking
task = r.blpop('queue', timeout=10)
if task:
    process(task[1])
```

#### Batch with Pipelining

```python
# ❌ BAD: Individual pushes
for task in tasks:
    r.rpush('queue', task)

# ✅ GOOD: Pipeline
pipe = r.pipeline()
for task in tasks:
    pipe.rpush('queue', task)
pipe.execute()
```

#### Trim Regularly

```python
# ✅ Keep manageable size
def maintain_feed(user_id):
    r.ltrim(f'feed:{user_id}', 0, 999)  # Keep 1000 items

# Run daily
schedule.every().day.do(maintain_all_feeds)
```

---

## Best Practices

### 1. Use Blocking Operations

```python
# ✅ DO: Efficient waiting
result = r.blpop(['queue1', 'queue2'], timeout=30)

# ❌ DON'T: Busy polling
while True:
    if r.llen('queue1') > 0:
        process(r.lpop('queue1'))
```

### 2. Handle Timeouts Properly

```python
# ✅ DO: Check timeout result
result = r.blpop('queue', timeout=5)
if result:
    process(result[1])
else:
    # Timeout - queue was empty
    log.warning("Queue empty")
```

### 3. Implement Retry Logic

```python
# ✅ DO: Failed tasks to DLQ
try:
    process(task)
except Exception:
    # Track retries
    retries = r.hincrby(f'task:{id}', 'retries', 1)
    if retries < 3:
        r.rpush('queue', task)  # Retry
    else:
        r.lpush('queue:failed', task)  # Give up
```

### 4. Monitor Queue Size

```bash
# ✅ DO: Watch queue backlog
LLEN task:queue     # Current size

# Alert if queue > 1000 items
if LLEN > 1000:
    alert_ops()
```

### 5. Set Consistent Naming

```
✅ GOOD PATTERNS:
task:queue
task:queue:failed
user:123:feed
notifications:user:456
undo:user:789
```

---

## Common Mistakes

### Mistake 1: Accessing Middle of Large List

```python
# ❌ WRONG: O(N) operation on large list
r.lrange('huge_list', 0, -1)
middle = list_data[500000]

# ✅ RIGHT: Use smaller ranges or pagination
first_100 = r.lrange('list', 0, 99)
```

### Mistake 2: Not Handling Timeout

```python
# ❌ WRONG: Ignores timeout
r.blpop('queue', 0)  # Waits forever

# ✅ RIGHT: Reasonable timeout
result = r.blpop('queue', timeout=60)
if not result:
    backoff()
```

### Mistake 3: Unbounded List Growth

```python
# ❌ WRONG: Never trimmed
r.rpush('feed:user', item)
# After years: millions of items!

# ✅ RIGHT: Maintain size
r.rpush('feed:user', item)
r.ltrim('feed:user', 0, 999)  # Keep 1000
```

### Mistake 4: Non-Atomic Multi-Operations

```python
# ❌ WRONG: Not atomic
task = r.lpop('queue')
r.lpush('processing', task)  # Gap between operations

# ✅ RIGHT: Use atomic operations or WATCH
pipe = r.pipeline()
pipe.lpop('queue')
pipe.lpush('processing', task)
pipe.execute()
```

### Mistake 5: Wrong Operation Direction

```python
# ❌ WRONG: FIFO becomes LIFO
r.lpush('queue', 'task1')
r.lpush('queue', 'task2')
r.rpop('queue')  # Gets task1, not task2!

# ✅ RIGHT: Consistent direction
r.rpush('queue', 'task1')
r.rpush('queue', 'task2')
r.lpop('queue')  # Gets task1 (FIFO)
```

---

## Next Steps

Explore other data structures:
- **[Sets](3-sets.md)** - Unique collections
- **[Hashes](4-hashes.md)** - Objects and profiles
- **[Sorted Sets](5-sorted-set.md)** - Rankings and leaderboards

Then build:
- Advanced queue systems (priority queue, DLQ)
- Real-time notifications
- Task processing frameworks
- Rate limiting systems

