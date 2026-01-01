# Redis Transactions

## Overview

Transactions execute a sequence of commands atomically. Redis guarantees all commands execute in order without interruption. Commands are queued with MULTI and executed with EXEC.

## Basic Transaction

### Commands Queued

```redis
# Start transaction
MULTI
# Output: OK

# Queue commands
SET key1 value1
# Output: QUEUED

SET key2 value2
# Output: QUEUED

GET key1
# Output: QUEUED

# Execute all at once
EXEC
# Output:
# 1) OK
# 2) OK
# 3) "value1"
```

### Discarding Transactions

```redis
MULTI
SET key "value"
# Output: QUEUED

DISCARD
# Output: OK

# Commands were not executed
GET key
# Output: (nil)
```

## Python Examples

### Basic Transaction

```python
import redis

r = redis.Redis()

# Create pipeline with transaction
with r.pipeline(transaction=True) as pipe:
    pipe.set('key1', 'value1')
    pipe.set('key2', 'value2')
    pipe.incr('counter')
    pipe.get('key1')
    
    results = pipe.execute()
    # Output: [True, True, 1, 'value1']
```

### MULTI/EXEC Explicitly

```python
import redis

r = redis.Redis()

# Manual MULTI/EXEC
pipe = r.pipeline()
pipe.multi()
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
results = pipe.execute()
```

### DISCARD Example

```python
import redis

r = redis.Redis()

pipe = r.pipeline(transaction=True)
pipe.set('key', 'value')

# Cancel transaction
pipe.reset()  # or use pipe.execute() without changes
```

## Atomic Operations

### Counter with Atomicity

```python
import redis

r = redis.Redis()

def atomic_increment():
    """Atomic counter increment"""
    with r.pipeline(transaction=True) as pipe:
        pipe.incr('counter')
        results = pipe.execute()
    return results[0]

# Safe from race conditions
for _ in range(100):
    atomic_increment()

# Counter is exactly 100
print(r.get('counter'))  # 100
```

### Transfer Money Atomically

```python
import redis

r = redis.Redis()

def transfer_funds(from_account, to_account, amount):
    """Atomically transfer funds between accounts"""
    with r.pipeline(transaction=True) as pipe:
        pipe.decrby(f'account:{from_account}', amount)
        pipe.incrby(f'account:{to_account}', amount)
        
        results = pipe.execute()
    return results

# Usage
r.set('account:alice', 1000)
r.set('account:bob', 500)

transfer_funds('alice', 'bob', 100)

print(r.get('account:alice'))  # 900
print(r.get('account:bob'))    # 600
```

## WATCH for Conditional Transactions

### Watching Keys

```python
import redis

r = redis.Redis()

def conditional_transaction():
    """Execute only if key hasn't changed"""
    while True:
        try:
            # Watch key for changes
            with r.pipeline(transaction=True) as pipe:
                pipe.watch('key1')
                
                # Read current value
                value = r.get('key1')
                
                # Prepare transaction
                pipe.multi()
                pipe.set('key1', int(value) + 1)
                
                # Execute - will fail if key1 changed
                pipe.execute()
                break
        except redis.WatchError:
            # Retry if key changed
            continue

# Usage
r.set('key1', 10)
conditional_transaction()
print(r.get('key1'))  # 11
```

### Practical WATCH Example

```python
import redis

r = redis.Redis()

def optimistic_update(key, transform_func):
    """Update with optimistic locking"""
    max_retries = 3
    retries = 0
    
    while retries < max_retries:
        try:
            with r.pipeline(transaction=True) as pipe:
                pipe.watch(key)
                
                # Read
                old_value = r.get(key)
                new_value = transform_func(old_value)
                
                # Prepare write
                pipe.multi()
                pipe.set(key, new_value)
                
                # Execute
                results = pipe.execute()
                return new_value
        except redis.WatchError:
            retries += 1
    
    raise Exception(f"Failed to update {key} after {max_retries} retries")

# Usage
r.set('config:version', '1')
new_version = optimistic_update('config:version', lambda v: str(int(v) + 1))
print(new_version)  # '2'
```

## Common Patterns

### Pattern 1: All-or-Nothing Update

```python
import redis

r = redis.Redis()

def update_user_atomically(user_id, updates):
    """Update multiple user fields atomically"""
    with r.pipeline(transaction=True) as pipe:
        for field, value in updates.items():
            pipe.hset(f'user:{user_id}', field, value)
        
        results = pipe.execute()
    
    return all(results)

# Usage
success = update_user_atomically(123, {
    'name': 'Alice',
    'email': 'alice@example.com',
    'updated_at': '2024-01-01'
})

if success:
    print("User updated atomically")
```

### Pattern 2: Conditional Counter

```python
import redis

r = redis.Redis()

def safe_counter_increment(key, max_value):
    """Increment counter only if below max"""
    while True:
        try:
            with r.pipeline(transaction=True) as pipe:
                pipe.watch(key)
                
                current = r.get(key)
                if not current:
                    current = 0
                else:
                    current = int(current)
                
                if current < max_value:
                    pipe.multi()
                    pipe.incr(key)
                    pipe.execute()
                    return True
                else:
                    return False
        except redis.WatchError:
            continue

# Usage
r.set('requests:user123', 0)
for i in range(5):
    if safe_counter_increment('requests:user123', 3):
        print(f"Request {i+1} allowed")
    else:
        print(f"Request {i+1} denied (limit exceeded)")
```

### Pattern 3: Race-Safe List Operations

```python
import redis

r = redis.Redis()

def atomic_list_append(list_key, item, max_length=100):
    """Append to list, keeping max length"""
    with r.pipeline(transaction=True) as pipe:
        pipe.lpush(list_key, item)
        pipe.ltrim(list_key, 0, max_length - 1)
        
        results = pipe.execute()
    
    return results

# Usage
r.delete('events')
atomic_list_append('events', {'id': 1, 'type': 'login'})
atomic_list_append('events', {'id': 2, 'type': 'purchase'})
```

## Error Handling

### Syntax Errors

```python
import redis

r = redis.Redis()

try:
    with r.pipeline(transaction=True) as pipe:
        pipe.set('key', 'value')
        pipe.incr('key')  # Will fail - key is not numeric
        pipe.execute()
except redis.ResponseError as e:
    print(f"Transaction error: {e}")
```

### WATCH Conflicts

```python
import redis

r = redis.Redis()

def handle_watch_error():
    """Handle WATCH conflicts gracefully"""
    max_retries = 3
    
    for attempt in range(max_retries):
        try:
            with r.pipeline(transaction=True) as pipe:
                pipe.watch('key')
                value = r.get('key')
                
                pipe.multi()
                pipe.set('key', int(value) + 1)
                pipe.execute()
                return True
        except redis.WatchError:
            if attempt == max_retries - 1:
                raise Exception("Too many retries")
            continue

handle_watch_error()
```

## Performance Considerations

### Time Complexity

```
MULTI:  O(1)
EXEC:   O(N) where N = number of queued commands
WATCH:  O(N) where N = number of watched keys
```

### Latency

```python
# Single roundtrip:
with r.pipeline(transaction=True) as pipe:
    for i in range(1000):
        pipe.set(f'key{i}', f'value{i}')
    pipe.execute()
# ~10-20ms for 1000 commands

# vs individual commands:
for i in range(1000):
    r.set(f'key{i}', f'value{i}')
# ~1-2s for 1000 commands
```

## Best Practices

### 1. Keep Transactions Short

```python
# Good: Transaction is minimal
with r.pipeline(transaction=True) as pipe:
    pipe.set('counter', 100)
    pipe.incr('total')
    pipe.execute()

# Avoid: Long operations in transaction
with r.pipeline(transaction=True) as pipe:
    # Fetch from DB
    data = fetch_large_dataset()  # SLOW!
    
    # Process
    processed = process_data(data)  # SLOW!
    
    # Update Redis
    pipe.set('result', processed)
    pipe.execute()
```

### 2. Watch Minimal Keys

```python
# Good: Watch only necessary keys
with r.pipeline(transaction=True) as pipe:
    pipe.watch('balance')  # Only watch balance
    balance = r.get('balance')
    
    pipe.multi()
    pipe.set('balance', int(balance) - 100)
    pipe.execute()

# Avoid: Watching too many keys
with r.pipeline(transaction=True) as pipe:
    pipe.watch('balance', 'total', 'daily_limit', 'account', 'user', ...)
```

### 3. Use Context Manager

```python
# Good: Automatic cleanup
with r.pipeline(transaction=True) as pipe:
    pipe.set('key', 'value')
    pipe.execute()

# Avoid: Manual management
pipe = r.pipeline(transaction=True)
pipe.set('key', 'value')
pipe.execute()
```

### 4. Handle Watch Failures

```python
# Good: Retry on WATCH failure
max_retries = 3
for attempt in range(max_retries):
    try:
        with r.pipeline(transaction=True) as pipe:
            pipe.watch('key')
            # Transaction code
            break
    except redis.WatchError:
        if attempt == max_retries - 1:
            raise

# Avoid: Ignoring WATCH failures
try:
    with r.pipeline(transaction=True) as pipe:
        pipe.watch('key')
        # Transaction code
except redis.WatchError:
    pass  # Silently ignore!
```

## Common Mistakes

### ❌ Mixing WATCH and MULTI Incorrectly

```python
# Bad: WATCH after MULTI
with r.pipeline(transaction=True) as pipe:
    pipe.multi()  # Starts transaction
    pipe.watch('key')  # Can't watch inside transaction!

# Good: WATCH before MULTI
with r.pipeline(transaction=True) as pipe:
    pipe.watch('key')  # Watch first
    pipe.multi()  # Then transaction
```

### ❌ Not Handling WATCH Errors

```python
# Bad: WATCH error ignored
try:
    with r.pipeline(transaction=True) as pipe:
        pipe.watch('key')
        pipe.execute()
except redis.WatchError:
    pass  # Transaction silently failed!

# Good: Handle and retry
try:
    with r.pipeline(transaction=True) as pipe:
        pipe.watch('key')
        pipe.execute()
except redis.WatchError:
    print("Conflict detected, retrying...")
    # Retry logic
```

### ❌ Assuming Atomicity Without Transaction

```python
# Bad: Not atomic
r.set('key1', 'value1')
r.set('key2', 'value2')
# Other clients can see inconsistent state

# Good: Atomic
with r.pipeline(transaction=True) as pipe:
    pipe.set('key1', 'value1')
    pipe.set('key2', 'value2')
    pipe.execute()
```

### ❌ Too Many Watched Keys

```python
# Bad: Watching too much
pipe.watch('key1', 'key2', 'key3', 'key4', 'key5')
# High chance of conflicts

# Good: Watch only what's necessary
pipe.watch('critical_key')
```

## Next Steps

- [Pipelining](8-pipeline.md) - Non-atomic bulk operations
- [Strings](6-strings.md) - Common transaction operations
- [Data Structures](../2-data-structure/1-intro.md) - Advanced patterns

## Resources

- **Transactions**: https://redis.io/topics/transactions/
- **WATCH**: https://redis.io/commands/watch/
- **MULTI/EXEC**: https://redis.io/commands/multi/

## Summary

- MULTI/EXEC ensures atomic execution of queued commands
- All commands execute in order without interruption
- WATCH enables optimistic locking (WATCH + MULTI + EXEC)
- Retry on WatchError when using WATCH
- Keep transactions short for better concurrency
- Context manager ensures proper cleanup
- Atomic operations prevent race conditions
