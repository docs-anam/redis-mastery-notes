# Lua Scripting - Atomic Server-Side Operations

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

Lua scripting allows executing complex operations atomically on the Redis server, eliminating race conditions and reducing round-trips.

### Key Advantages

✅ **Atomicity**: Entire script runs without interruption
✅ **Efficiency**: Single round-trip for complex operations
✅ **Consistency**: Eliminates race conditions
✅ **Flexibility**: Complex logic server-side

### When to Use Scripts

✅ **Good For**:
- Conditional updates (check then set)
- Multi-step operations
- Rate limiting
- Atomic counters
- Complex transactions
- Distributed locks

❌ **Not Good For**:
- Simple operations (use commands instead)
- Long-running operations (blocking)
- Non-deterministic logic
- Accessing external APIs

---

## Core Concepts

### Atomicity Guarantee

```
Without Script:              With Script:
1. GET key1                  1. EVAL script
2. Check value         →     2. Atomic execution
3. SET key2                  3. All-or-nothing
4. INCR key3           
(Race conditions!)           (Safe from race!)
```

### KEYS vs ARGV

Understanding the distinction:

```python
# Keys: Can be shared across replicas/cluster
redis.eval(script, numkeys, key1, key2, arg1, arg2)

# In Lua:
# KEYS[1] = key1
# KEYS[2] = key2
# ARGV[1] = arg1
# ARGV[2] = arg2
```

### Script Caching

Scripts are cached by SHA1 hash:

```
1. Client sends script
2. Redis computes SHA1
3. Caches the script
4. Returns SHA1
5. Subsequent calls use SHA1 (more efficient)
```

---

## Commands Reference

### EVAL
Execute Lua script.

```redis
EVAL script numkeys key [key ...] arg [arg ...]
# Returns: Result from script
```

```python
import redis

r = redis.Redis(host='localhost', port=6379)

script = """
local current = redis.call('GET', KEYS[1])
if current == false then
    redis.call('SET', KEYS[1], ARGV[1])
    return 'SET'
else
    return 'EXISTS'
end
"""

result = r.eval(script, 1, 'mykey', 'myvalue')
print(result)  # 'SET' or 'EXISTS'
```

### EVALSHA
Execute cached script by SHA1.

```redis
EVALSHA sha1 numkeys key [key ...] arg [arg ...]
```

```python
# First execution
sha1 = r.script_load(script)

# Subsequent executions (faster)
result = r.evalsha(sha1, 1, 'mykey', 'newvalue')
```

### SCRIPT LOAD
Cache a script without executing.

```redis
SCRIPT LOAD script
# Returns: SHA1 of script
```

```python
sha1 = r.script_load(script)
print(f"Script SHA1: {sha1}")
```

### SCRIPT EXISTS
Check if script is cached.

```redis
SCRIPT EXISTS sha1 [sha1 ...]
# Returns: Array of 0/1 values
```

```python
exists = r.script_exists(sha1)
print(f"Script cached: {exists}")
```

### SCRIPT FLUSH
Clear script cache.

```redis
SCRIPT FLUSH
```

```python
r.script_flush()  # Clear all scripts
```

---

## Practical Examples

### Example 1: Rate Limiting

```python
import redis
from datetime import datetime

class RateLimiter:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
        
        self.script = """
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        
        local current = redis.call('GET', key)
        if current == nil then
            redis.call('SET', key, 1)
            redis.call('EXPIRE', key, window)
            return 1
        elseif tonumber(current) < limit then
            redis.call('INCR', key)
            return tonumber(redis.call('GET', key))
        else
            return -1  -- Rate limited
        end
        """
        
        self.sha1 = self.r.script_load(self.script)
    
    def is_allowed(self, client_id, limit=10, window=60):
        """Check if client is within rate limit"""
        key = f'ratelimit:{client_id}'
        result = self.r.evalsha(self.sha1, 1, key, limit, window)
        return result > 0

# Usage
limiter = RateLimiter()

# Simulate requests
for i in range(12):
    allowed = limiter.is_allowed('user:123')
    print(f"Request {i+1}: {'✅ Allowed' if allowed else '❌ Blocked'}")
```

### Example 2: Atomic Counter with Threshold

```python
import redis

class ThresholdCounter:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
        
        self.script = """
        local key = KEYS[1]
        local threshold = tonumber(ARGV[1])
        local increment = tonumber(ARGV[2])
        
        local current = tonumber(redis.call('GET', key) or 0)
        local new_value = current + increment
        
        if new_value >= threshold then
            redis.call('SET', key, 0)  -- Reset on threshold
            return 'THRESHOLD_REACHED'
        else
            redis.call('SET', key, new_value)
            return tostring(new_value)
        end
        """
    
    def increment(self, key, threshold, delta=1):
        """Atomically increment and check threshold"""
        result = self.r.eval(self.script, 1, key, threshold, delta)
        return result.decode() if isinstance(result, bytes) else result

# Usage
counter = ThresholdCounter()

for i in range(15):
    result = counter.increment('requests', 10, 1)
    print(f"Count: {result}")
```

### Example 3: Distributed Lock

```python
import redis
import uuid
from datetime import datetime, timedelta

class DistributedLock:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
        self.lock_id = str(uuid.uuid4())
        
        # Script to acquire lock safely
        self.acquire_script = """
        if redis.call('EXISTS', KEYS[1]) == 0 then
            redis.call('SET', KEYS[1], ARGV[1])
            redis.call('EXPIRE', KEYS[1], ARGV[2])
            return 1
        else
            return 0
        end
        """
        
        # Script to release lock (only if we own it)
        self.release_script = """
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            redis.call('DEL', KEYS[1])
            return 1
        else
            return 0
        end
        """
    
    def acquire(self, resource_name, timeout=30):
        """Acquire distributed lock"""
        key = f'lock:{resource_name}'
        acquired = self.r.eval(
            self.acquire_script, 
            1, 
            key, 
            self.lock_id, 
            timeout
        )
        return acquired == 1
    
    def release(self, resource_name):
        """Release distributed lock"""
        key = f'lock:{resource_name}'
        released = self.r.eval(self.release_script, 1, key, self.lock_id)
        return released == 1

# Usage
lock = DistributedLock()

if lock.acquire('critical_resource', timeout=30):
    try:
        print("Lock acquired, performing operation...")
        # Critical section
    finally:
        lock.release('critical_resource')
        print("Lock released")
else:
    print("Could not acquire lock")
```

### Example 4: Conditional Update with Multiple Keys

```python
import redis
import json

class TransferService:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
        
        self.script = """
        local from_key = KEYS[1]
        local to_key = KEYS[2]
        local amount = tonumber(ARGV[1])
        
        local from_balance = tonumber(redis.call('GET', from_key) or 0)
        local to_balance = tonumber(redis.call('GET', to_key) or 0)
        
        if from_balance < amount then
            return {0, 'INSUFFICIENT_BALANCE'}
        end
        
        redis.call('DECRBY', from_key, amount)
        redis.call('INCRBY', to_key, amount)
        
        return {1, from_balance - amount, to_balance + amount}
        """
    
    def transfer(self, from_account, to_account, amount):
        """Atomically transfer funds"""
        from_key = f'balance:{from_account}'
        to_key = f'balance:{to_account}'
        
        result = self.r.eval(self.script, 2, from_key, to_key, amount)
        
        if isinstance(result, list):
            if result[0] == 1:
                return {
                    'success': True,
                    'from_balance': result[1],
                    'to_balance': result[2]
                }
            else:
                return {
                    'success': False,
                    'error': result[1]
                }

# Usage
service = TransferService()

# Set initial balances
r = redis.Redis()
r.set('balance:account1', 1000)
r.set('balance:account2', 500)

# Perform transfer
result = service.transfer('account1', 'account2', 100)
print(json.dumps(result, indent=2))
```

---

## Real-World Patterns

### Pattern 1: Leaky Bucket Rate Limiter

```python
import redis
from datetime import datetime

class LeakyBucketLimiter:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
        
        self.script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local leak_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        
        local bucket = redis.call('HGETALL', key)
        if #bucket == 0 then
            redis.call('HSET', key, 'tokens', capacity, 'last_refill', now)
            return 1
        end
        
        local tokens = tonumber(bucket[2])
        local last_refill = tonumber(bucket[4])
        local elapsed = now - last_refill
        
        tokens = math.min(capacity, tokens + (elapsed * leak_rate))
        
        if tokens >= 1 then
            redis.call('HSET', key, 'tokens', tokens - 1, 'last_refill', now)
            return 1
        else
            return 0
        end
        """
    
    def is_allowed(self, client_id, capacity=100, leak_rate=10):
        """Check rate limit with leaky bucket algorithm"""
        key = f'bucket:{client_id}'
        now = datetime.now().timestamp()
        
        allowed = self.r.eval(
            self.script, 
            1, 
            key, 
            capacity, 
            leak_rate, 
            now
        )
        return allowed == 1
```

### Pattern 2: Inventory Management

```python
import redis

class InventoryManager:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
        
        self.script = """
        local product_key = KEYS[1]
        local order_key = KEYS[2]
        local quantity = tonumber(ARGV[1])
        local order_id = ARGV[2]
        
        local available = tonumber(redis.call('GET', product_key) or 0)
        
        if available >= quantity then
            redis.call('DECRBY', product_key, quantity)
            redis.call('INCR', order_key)
            return 'RESERVED'
        else
            return 'OUT_OF_STOCK'
        end
        """
    
    def reserve_inventory(self, product_id, quantity, order_id):
        """Atomically reserve inventory"""
        product_key = f'inventory:{product_id}'
        order_key = f'order:{order_id}'
        
        result = self.r.eval(
            self.script, 
            2, 
            product_key, 
            order_key, 
            quantity, 
            order_id
        )
        return result.decode()
```

---

## Performance Characteristics

### Execution Speed

```
Operation Type           Time          Throughput
─────────────────────────────────────────────────
Simple command          0.1ms          ~10k ops/sec
Small script (5 commands) 0.5ms         ~2k ops/sec
Medium script (20 commands) 2ms         ~500 ops/sec
Large script (100+ commands) 10ms       ~100 ops/sec
```

### Memory Usage

```
Script Size    Cached Memory    Max Scripts    Total Memory
─────────────────────────────────────────────────────────
1 KB           ~2 KB            10,000         ~20 MB
10 KB          ~20 KB           1,000          ~20 MB
100 KB         ~200 KB          100            ~20 MB
```

---

## Best Practices

### DO: Design Deterministic Scripts

```python
# ✅ GOOD: Deterministic (same input = same output)
script = """
local value = tonumber(ARGV[1])
return value * 2
"""

# ❌ BAD: Non-deterministic (uses current time)
script = """
return redis.call('TIME')
"""
```

### DO: Keep Scripts Short

```python
# ✅ GOOD: Focused script
script = """
if redis.call('EXISTS', KEYS[1]) == 0 then
    redis.call('SET', KEYS[1], ARGV[1])
    return 1
end
return 0
"""

# ❌ BAD: Long complex script
# (Consider client-side logic instead)
```

### DO: Use EVALSHA for Repeated Scripts

```python
# ✅ GOOD: Cache and reuse
sha1 = r.script_load(script)
result = r.evalsha(sha1, 1, key)
result = r.evalsha(sha1, 1, key)

# ❌ BAD: Send entire script each time
result = r.eval(script, 1, key)
result = r.eval(script, 1, key)
```

### DO: Handle Script Errors

```python
import redis

try:
    result = r.eval(script, 1, key)
except redis.ResponseError as e:
    print(f"Script error: {e}")
    # Handle error (retry, use fallback, etc.)
```

### DON'T: Block with Long Scripts

```python
# ❌ BAD: Blocking Redis for 5 seconds
script = """
local result = 0
for i = 1, 5000000 do
    result = result + i
end
return result
"""

# ✅ GOOD: Quick execution
script = """
return redis.call('INCR', KEYS[1])
"""
```

### DON'T: Use Script for Non-Redis Calls

```python
# ❌ BAD: Trying to call external API
script = """
local response = http.get('http://api.example.com')
return response
"""

# ✅ GOOD: Redis-only operations
script = """
return redis.call('GET', KEYS[1])
"""
```

---

## Common Mistakes

### Mistake 1: Blocking Operations

**Problem**: Long-running scripts block all clients
```python
# ❌ 10-second computation blocks Redis
script = """
for i = 1, 10000000 do
    -- Heavy computation
end
"""
```

**Solution**: Keep scripts under 10ms
```python
# ✅ Quick operations
script = """
return redis.call('INCR', KEYS[1])
"""
```

### Mistake 2: Ignoring EVALSHA Fallback

**Problem**: Script not cached on replica
```python
# ❌ Fails if script not in cache
r.evalsha(sha1, 1, key)
```

**Solution**: Handle both EVAL and EVALSHA
```python
# ✅ Fallback to EVAL
try:
    return r.evalsha(sha1, 1, key)
except redis.ResponseError:
    return r.eval(script, 1, key)
```

### Mistake 3: Non-Deterministic Logic

**Problem**: Different output for same input
```python
# ❌ Different result each time
script = """
return redis.call('TIME')[1]
"""

# ❌ Depends on system state
script = """
return math.random()
"""
```

**Solution**: Use only deterministic operations
```python
# ✅ Deterministic
script = """
return redis.call('GET', KEYS[1])
"""
```

### Mistake 4: Complex Error Handling

**Problem**: Poor error handling in script
```python
# ❌ No error handling
script = """
local value = redis.call('GET', KEYS[1])
return tonumber(value) + 1
"""
```

**Solution**: Validate in script
```python
# ✅ Proper validation
script = """
local value = redis.call('GET', KEYS[1])
if value == false then
    return {0, 'KEY_NOT_FOUND'}
end
return {1, tonumber(value) + 1}
"""
```

### Mistake 5: Incorrect KEYS/ARGV Usage

**Problem**: Using ARGV for keys
```python
# ❌ WRONG: Key as argument
r.eval(script, 0, mykey)  # numkeys=0!

# ❌ WRONG: Mixing purposes
r.eval(script, 1, key, anotherkey)
```

**Solution**: Correct ordering
```python
# ✅ CORRECT: Keys first, args last
r.eval(script, 2, key1, key2, arg1, arg2)
```

### Mistake 6: Not Testing on Replica

**Problem**: Script works on master, fails on replica
```python
# ❌ Test only on primary
# Replica may not have script cached
```

**Solution**: Test replication
```python
# ✅ Verify on replicas too
# Use EVALSHA with fallback to EVAL
```

---

## Next Steps

1. **Explore [Replication](4-replication.md)** for script distribution
2. **Learn [Cluster](6-cluster.md)** for script execution in cluster
3. **Study [Performance](../4-performance/1-intro.md)** for optimization
4. **Set up [Monitoring](../1-basics/11-server-information.md)** for script performance

## Resources

- [Redis Lua Scripting](https://redis.io/commands/eval)
- [Lua Documentation](https://www.lua.org/manual/5.1/)
- [Redis Scripting Debugging](https://redis.io/topics/ldb)
- [Script Atomicity Guide](https://redis.io/topics/transactions)
