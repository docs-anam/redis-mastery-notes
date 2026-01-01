# Redis Pipelining

## Overview

Pipelining allows sending multiple commands to Redis without waiting for replies. Significantly improves throughput by reducing network round trips.

## How Pipelining Works

### Traditional Request-Response

```
Client      Redis
  |          |
  |-- GET key1 -->|
  |          |
  |<-- result1 --|
  |          |
  |-- GET key2 -->|
  |          |
  |<-- result2 --|
  |          |
```

Multiple round trips = High latency

### With Pipelining

```
Client      Redis
  |          |
  |-- GET key1 -->|
  |-- GET key2 -->|
  |-- GET key3 -->|
  |          |
  |<-- result1 --|
  |<-- result2 --|
  |<-- result3 --|
  |          |
```

Single round trip = Low latency

## Basic Pipelining

### Python Example

```python
import redis

r = redis.Redis()

# Create a pipeline
pipe = r.pipeline()

# Queue multiple commands
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
pipe.set('key3', 'value3')
pipe.get('key1')
pipe.get('key2')
pipe.get('key3')

# Execute all at once
results = pipe.execute()
# Output: [True, True, True, 'value1', 'value2', 'value3']
```

### Using `with` Statement

```python
import redis

r = redis.Redis()

# Context manager automatically executes
with r.pipeline() as pipe:
    pipe.set('key1', 'value1')
    pipe.set('key2', 'value2')
    pipe.incr('counter')
    pipe.get('counter')
    
    results = pipe.execute()
    print(results)
    # [True, True, 1, '1']
```

## Performance Impact

### Benchmark Example

```python
import redis
import time

r = redis.Redis()

# Without pipeline (10,000 requests)
start = time.time()
for i in range(10000):
    r.set(f'key{i}', f'value{i}')
elapsed_slow = time.time() - start
print(f"Without pipeline: {elapsed_slow:.2f}s")  # ~1.5-2s

# With pipeline (batched)
start = time.time()
pipe = r.pipeline()
for i in range(10000):
    pipe.set(f'key{i}', f'value{i}')
    if i % 1000 == 0:  # Batch every 1000
        pipe.execute()
        pipe = r.pipeline()
elapsed_fast = time.time() - start
print(f"With pipeline: {elapsed_fast:.2f}s")  # ~0.2-0.3s

print(f"Speedup: {elapsed_slow/elapsed_fast:.1f}x")  # ~5-7x faster
```

## Practical Examples

### Example 1: Bulk Insert

```python
import redis
import json

r = redis.Redis()

def bulk_insert_users(users):
    """Insert multiple users using pipeline"""
    with r.pipeline() as pipe:
        for user in users:
            user_key = f"user:{user['id']}"
            user_json = json.dumps(user)
            pipe.set(user_key, user_json)
        
        results = pipe.execute()
    
    return len(results)

# Usage
users = [
    {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'},
    {'id': 2, 'name': 'Bob', 'email': 'bob@example.com'},
    {'id': 3, 'name': 'Charlie', 'email': 'charlie@example.com'},
]

inserted = bulk_insert_users(users)
print(f"Inserted {inserted} users")
```

### Example 2: Increment Multiple Counters

```python
import redis

r = redis.Redis()

def increment_user_stats(user_id, stats):
    """Atomically increment multiple stats"""
    with r.pipeline() as pipe:
        for stat_name, value in stats.items():
            key = f"stats:{user_id}:{stat_name}"
            pipe.incrby(key, value)
        
        results = pipe.execute()
    
    return results

# Usage
stats = {
    'page_views': 5,
    'clicks': 2,
    'purchases': 1,
}

results = increment_user_stats(123, stats)
print(f"New stats: {results}")
```

### Example 3: Cache Warm-up

```python
import redis
import requests

r = redis.Redis(decode_responses=True)

def warm_cache(product_ids):
    """Pre-populate cache with product data"""
    with r.pipeline() as pipe:
        for product_id in product_ids:
            # Fetch product
            product = fetch_product(product_id)
            
            # Queue cache set
            key = f"product:{product_id}"
            pipe.setex(key, 3600, json.dumps(product))
        
        pipe.execute()

def fetch_product(product_id):
    """Fetch from database"""
    # Simulate DB call
    return {'id': product_id, 'name': f'Product {product_id}'}

# Usage
warm_cache([1, 2, 3, 4, 5])
print("Cache warmed!")
```

### Example 4: Batch Operations

```python
import redis

r = redis.Redis()

def batch_delete(pattern, batch_size=1000):
    """Delete keys matching pattern in batches"""
    cursor = 0
    deleted = 0
    
    while True:
        # Scan for keys
        cursor, keys = r.scan(cursor, match=pattern, count=batch_size)
        
        if keys:
            # Delete in pipeline
            with r.pipeline() as pipe:
                for key in keys:
                    pipe.delete(key)
                pipe.execute()
                deleted += len(keys)
        
        if cursor == 0:
            break
    
    return deleted

# Usage
deleted = batch_delete('temp:*')
print(f"Deleted {deleted} keys")
```

## Transaction vs Pipeline

### Difference

```python
import redis

r = redis.Redis()

# Pipeline: Send multiple commands efficiently
# NO atomicity guarantee
pipe = r.pipeline()
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
results = pipe.execute()  # Other clients can interleave

# Transaction: Atomic execution
# Guarantee of atomicity
pipe = r.pipeline(transaction=True)  # MULTI/EXEC
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
results = pipe.execute()  # Atomic, no interleaving
```

### Choosing Between Them

| Use Case | Pipeline | Transaction |
|----------|----------|-------------|
| Bulk insert | ✓ | - |
| Counter increment | - | ✓ |
| Cache warmup | ✓ | - |
| Atomic updates | - | ✓ |
| Performance critical | ✓ | - |
| Must prevent races | - | ✓ |

## Advanced Patterns

### Pattern 1: Conditional Pipeline

```python
import redis

r = redis.Redis()

def conditional_update(user_id, new_balance):
    """Update balance only if sufficient funds"""
    with r.pipeline(transaction=True) as pipe:
        try:
            pipe.watch(f"user:{user_id}:balance")
            current = r.get(f"user:{user_id}:balance")
            
            if int(current) >= 100:  # Sufficient balance
                pipe.multi()
                pipe.decrby(f"user:{user_id}:balance", new_balance)
                pipe.execute()
                return True
            else:
                return False
        except redis.WatchError:
            return False  # Conflict detected
```

### Pattern 2: Dynamic Pipeline

```python
import redis

r = redis.Redis()

def smart_pipeline(operations, batch_size=100):
    """Pipeline with dynamic batching"""
    results = []
    
    for i in range(0, len(operations), batch_size):
        batch = operations[i:i+batch_size]
        
        with r.pipeline() as pipe:
            for op_type, *args in batch:
                if op_type == 'set':
                    pipe.set(*args)
                elif op_type == 'incr':
                    pipe.incr(*args)
                elif op_type == 'delete':
                    pipe.delete(*args)
            
            batch_results = pipe.execute()
            results.extend(batch_results)
    
    return results

# Usage
operations = [
    ('set', 'key1', 'value1'),
    ('incr', 'counter'),
    ('delete', 'temp_key'),
]

results = smart_pipeline(operations)
```

### Pattern 3: Error Handling

```python
import redis

r = redis.Redis()

def safe_pipeline(commands):
    """Pipeline with error handling"""
    try:
        with r.pipeline() as pipe:
            for cmd_name, *args in commands:
                getattr(pipe, cmd_name)(*args)
            
            results = pipe.execute()
            return results, None
    except redis.RedisError as e:
        return None, str(e)

# Usage
commands = [
    ('set', 'key1', 'value1'),
    ('get', 'key1'),
    ('incr', 'counter'),
]

results, error = safe_pipeline(commands)
if error:
    print(f"Pipeline error: {error}")
else:
    print(f"Results: {results}")
```

## Best Practices

### 1. Batch Appropriately

```python
# Good: Batch size 100-1000
with r.pipeline() as pipe:
    for i in range(500):
        pipe.set(f'key{i}', f'value{i}')
    pipe.execute()

# Bad: Too large (memory overhead)
with r.pipeline() as pipe:
    for i in range(100000):
        pipe.set(f'key{i}', f'value{i}')
    pipe.execute()  # Uses lots of memory

# Better: Multiple pipelines
for batch_start in range(0, 100000, 1000):
    with r.pipeline() as pipe:
        for i in range(batch_start, batch_start + 1000):
            pipe.set(f'key{i}', f'value{i}')
        pipe.execute()
```

### 2. Use Context Manager

```python
# Good: Automatic cleanup
with r.pipeline() as pipe:
    pipe.set('key', 'value')
    pipe.execute()

# Avoid: Manual management
pipe = r.pipeline()
pipe.set('key', 'value')
pipe.execute()  # What if error occurs?
```

### 3. Handle Exceptions

```python
import redis

r = redis.Redis()

try:
    with r.pipeline() as pipe:
        pipe.set('key', 'value')
        pipe.incr('non_numeric')  # Will fail
        pipe.execute()
except redis.ResponseError as e:
    print(f"Pipeline error: {e}")
```

### 4. Monitor Pipeline Performance

```python
import redis
import time

r = redis.Redis()

def timed_pipeline(commands):
    """Pipeline with timing"""
    start = time.time()
    
    with r.pipeline() as pipe:
        for cmd_name, *args in commands:
            getattr(pipe, cmd_name)(*args)
        
        results = pipe.execute()
    
    elapsed = time.time() - start
    print(f"Pipeline ({len(commands)} cmds): {elapsed*1000:.2f}ms")
    
    return results
```

## Common Mistakes

### ❌ Too Large Pipelines

```python
# Bad: 100,000+ commands at once
pipe = r.pipeline()
for i in range(100000):
    pipe.set(f'key{i}', f'value{i}')
pipe.execute()  # High memory usage

# Good: Batch them
for batch in range(0, 100000, 1000):
    with r.pipeline() as pipe:
        for i in range(batch, batch+1000):
            pipe.set(f'key{i}', f'value{i}')
        pipe.execute()
```

### ❌ Not Using Context Manager

```python
# Bad: Exception during execute
pipe = r.pipeline()
pipe.set('key', 'value')
# What if error here?
pipe.execute()

# Good: Guaranteed cleanup
with r.pipeline() as pipe:
    pipe.set('key', 'value')
    pipe.execute()
```

### ❌ Assuming Atomicity Without MULTI

```python
# Bad: No atomicity guarantee
pipe = r.pipeline()
pipe.incr('counter')
pipe.execute()  # Other clients can interleave

# Good: Use transaction
pipe = r.pipeline(transaction=True)
pipe.incr('counter')
pipe.execute()  # Atomic
```

## Next Steps

- [Transactions](9-transaction.md) - MULTI/EXEC for atomicity
- [Strings](6-strings.md) - Most common pipeline operations
- [Performance](../2-data-structure/1-intro.md) - Optimization techniques

## Resources

- **Pipeline Documentation**: https://redis.io/topics/pipelining/
- **Performance**: https://redis.io/docs/management/optimization/

## Summary

- Pipeline sends multiple commands in one request
- Reduces network latency dramatically (5-7x faster)
- Not atomic unless using transaction=True
- Batch size: 100-1000 commands optimal
- Use context manager for cleanup
- Best for bulk operations without atomicity needs
