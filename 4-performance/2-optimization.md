# Optimization - Application-Level Performance

## Overview

Application-level optimizations provide 5-10x performance improvements with minimal infrastructure changes. Focus on reducing round-trips and using efficient patterns.

## Core Optimization Techniques

### 1. Pipelining

Send multiple commands without waiting for responses:

```python
import redis

r = redis.Redis(host='localhost', port=6379)

# ❌ BAD: 1000 round-trips
for i in range(1000):
    r.set(f'key:{i}', f'value:{i}')
# ~100-200ms

# ✅ GOOD: 1 round-trip
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
# ~10-20ms (10x faster!)
```

**Benefits**: Reduces latency from O(n) to O(1)
**Typical improvement**: 5-10x for batch operations

### 2. Connection Pooling

Reuse connections instead of creating new ones:

```python
# ✅ GOOD: Pool of connections
pool = redis.ConnectionPool(host='localhost', port=6379, max_connections=10)
r = redis.Redis(connection_pool=pool)

# Multiple threads/coroutines share the pool
# No connection overhead per operation
```

**Benefits**: Reduces connection setup time
**Typical improvement**: 20-30% for high-concurrency scenarios

### 3. Batch Operations

Group related operations:

```python
# ❌ BAD: Individual operations
r.set('counter:1', 0)
r.set('counter:2', 0)
r.set('counter:3', 0)

# ✅ GOOD: MSET batch
r.mset({'counter:1': 0, 'counter:2': 0, 'counter:3': 0})

# Even better: Pipeline with batch
pipe = r.pipeline()
pipe.set('counter:1', 0)
pipe.set('counter:2', 0)
pipe.set('counter:3', 0)
pipe.execute()
```

**Benefits**: Reduces overhead per key
**Typical improvement**: 2-3x for write-heavy workloads

### 4. Lua Scripting

Reduce round-trips by moving logic to server:

```python
# ❌ BAD: 2 round-trips
r.incr('counter')
if r.get('counter') > 1000:
    r.delete('counter')

# ✅ GOOD: 1 round-trip (atomic!)
script = """
redis.call('INCR', KEYS[1])
if tonumber(redis.call('GET', KEYS[1])) > 1000 then
    redis.call('DEL', KEYS[1])
end
"""
r.eval(script, 1, 'counter')
```

**Benefits**: Atomicity + fewer round-trips
**Typical improvement**: 3-5x for complex operations

### 5. Data Structure Selection

Choose the optimal data structure:

```
Operation                   Best Structure      Time
──────────────────────────────────────────────────
Simple value                String              O(1)
Unique members              Set                 O(1)
Ordered ranking             Sorted Set          O(log N)
Time-series events          Stream              O(1)
Queue/Stack                 List                O(1)
Object fields               Hash                O(1)
```

```python
# ❌ BAD: Storing object as JSON string
user = json.dumps({'name': 'Alice', 'email': 'alice@example.com'})
r.set('user:123', user)
# Must deserialize entire object

# ✅ GOOD: Use hash
r.hset('user:123', mapping={'name': 'Alice', 'email': 'alice@example.com'})
# Direct field access
```

**Benefits**: Faster access, lower memory
**Typical improvement**: 2-3x + 30% memory savings

---

## Performance Comparison

| Technique | Speedup | Implementation | Use Case |
|-----------|---------|-----------------|----------|
| Pipelining | 5-10x | Easy | Batch operations |
| Connection Pool | 1.2-1.3x | Trivial | High concurrency |
| Lua Scripts | 3-5x | Medium | Complex operations |
| MGET/MSET | 2-3x | Easy | Bulk reads/writes |
| Proper structures | 2-3x | Medium | New code |

---

## Best Practices

### DO: Pipeline Batch Operations
```python
# ✅ GOOD
pipe = r.pipeline()
for user_id in user_ids:
    pipe.get(f'user:{user_id}')
results = pipe.execute()
```

### DO: Use MGET for Multiple Keys
```python
# ✅ GOOD: Single round-trip
values = r.mget(['key1', 'key2', 'key3'])
```

### DON'T: Create Connection per Request
```python
# ❌ BAD: Connection overhead
for i in range(100):
    r = redis.Redis(host='localhost')
    r.set(f'key:{i}', f'value:{i}')

# ✅ GOOD: Reuse connection
r = redis.Redis(host='localhost')
for i in range(100):
    r.set(f'key:{i}', f'value:{i}')
```

---

## Common Mistakes

### Mistake 1: Not Using Pipelining
**Problem**: N round-trips for N operations
**Solution**: Use pipeline() for batches

### Mistake 2: Large Value Sizes
**Problem**: Memory bloat and network overhead
**Solution**: Compress or split large values

### Mistake 3: Blocking Operations
**Problem**: Thread pool starvation
**Solution**: Use async/await or separate thread pool

### Mistake 4: Wrong Encoding
**Problem**: JSON overhead vs binary
**Solution**: Use efficient serialization (msgpack, protobuf)

### Mistake 5: Ignoring Connection Limits
**Problem**: Resource exhaustion
**Solution**: Configure pool size based on concurrency

### Mistake 6: No Retry Logic
**Problem**: Transient failures cause crashes
**Solution**: Implement exponential backoff

---

## Next Steps

1. **Learn [Benchmarking](3-benchmarking.md)** to measure improvements
2. **Explore [Scaling](4-scaling.md)** for distribution
3. **Study [Monitoring](5-monitoring.md)** for production

## Resources

- [Redis Pipelining](https://redis.io/topics/pipelining)
- [Connection Pooling Patterns](https://redis.io/docs/manual/client-side-caching/)
- [Lua Script Performance](https://redis.io/commands/eval)
