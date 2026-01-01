# Redis Strings

## Overview

Strings are the most fundamental Redis data type. A Redis string is a sequence of bytes - the simplest data structure Redis offers. Unlike many key-value stores, Redis strings can hold binary data (not just text).

## String Basics

### Key Characteristics

- **Binary-safe**: Can store any byte sequence (text, images, serialized objects)
- **Maximum size**: 512MB per key
- **Atomic operations**: String operations are atomic
- **Indexed access**: Can work with specific bytes in a string
- **Numeric operations**: Can increment/decrement if content is numeric

### Creating and Retrieving Strings

```redis
# Set a string value
SET key "Hello World"

# Get the string value
GET key
# Output: "Hello World"

# Get multiple values
MGET key1 key2 key3

# Get with condition (only if not exists)
SETNX key "value"  # Only sets if key doesn't exist
```

## Basic String Commands

### Setting Values

```redis
# Simple set
SET key "value"

# Set with expiration (seconds)
SET key "value" EX 60    # Expires in 60 seconds
SET key "value" PX 60000 # Expires in 60000 milliseconds

# Set only if key exists
SET key "value" XX

# Set only if key doesn't exist
SET key "value" NX

# Get old value and set new one
GETSET key "newvalue"   # Returns old value, sets new one

# Set multiple values
MSET key1 "value1" key2 "value2" key3 "value3"

# Set multiple only if none exist
MSETNX key1 "value1" key2 "value2"
```

### Getting Values

```redis
# Get single value
GET key

# Get multiple values
MGET key1 key2 key3
# Output: ["value1", "value2", "value3"]

# Get and delete
GETDEL key  # Returns value and deletes

# Get value and set new expiration
GETEX key EX 60  # Returns value, sets 60 second expiration

# Get substring
GETRANGE key 0 4   # Get bytes 0-4
GETRANGE key -3 -1 # Get last 3 bytes
```

## String Length Operations

```redis
# Get length of string (in bytes)
STRLEN key

# Example:
SET msg "Hello"     # 5 bytes
STRLEN msg          # Returns: 5

# For UTF-8:
SET emoji "👍"      # 4 bytes (UTF-8 encoding)
STRLEN emoji        # Returns: 4
```

## Numeric Operations

### Increment and Decrement

```redis
# Set initial counter
SET counter 10

# Increment by 1
INCR counter           # Now 11

# Decrement by 1
DECR counter           # Now 10

# Increment by N
INCRBY counter 5       # Now 15

# Decrement by N
DECRBY counter 3       # Now 12

# Increment as float
INCRBYFLOAT counter 0.5  # Now 12.5

# Examples:
SET page_views 0
INCR page_views        # 1
INCR page_views        # 2
INCR page_views        # 3
GET page_views         # "3"
```

### Performance of Numeric Operations

```python
# Why use INCR instead of GET/SET?
import redis
import time

r = redis.Redis()

# Method 1: GET and SET (slow, not atomic)
def increment_slow():
    val = int(r.get('counter'))
    r.set('counter', val + 1)

# Method 2: INCR (fast, atomic)
def increment_fast():
    r.incr('counter')

# Benchmark
start = time.time()
for _ in range(10000):
    increment_fast()
end = time.time()
print(f"INCR time: {end - start:.3f}s")  # ~0.05s

start = time.time()
for _ in range(10000):
    increment_slow()
end = time.time()
print(f"GET/SET time: {end - start:.3f}s")  # ~0.15s
```

## Append and Bit Operations

### Append to Strings

```redis
# Initialize
SET name "Hello"

# Append text
APPEND name " World"

# Result
GET name  # "Hello World"

# Append returns new length
APPEND name "!"  # Returns: 12
```

### Bit Operations

```redis
# Get bit value at position
GETBIT key 0

# Set bit value at position
SETBIT key 0 1

# Count set bits in string
BITCOUNT key
BITCOUNT key 0 1   # Count in range of bytes

# Example: User visited days
SET visited_days "0"
SETBIT visited_days 0 1  # Day 0: visited
SETBIT visited_days 2 1  # Day 2: visited
BITCOUNT visited_days    # Visited 2 days
```

## Real-World Examples

### Example 1: Session Storage

```python
import redis
import json
from datetime import datetime

r = redis.Redis(decode_responses=True)

# Store session
session_data = {
    'user_id': 123,
    'username': 'alice',
    'login_time': datetime.now().isoformat()
}

# Store as JSON string
session_key = f"session:{session_id}"
r.setex(session_key, 3600, json.dumps(session_data))  # 1 hour

# Retrieve session
session = json.loads(r.get(session_key))
```

### Example 2: Page View Counter

```python
import redis

r = redis.Redis()

# Increment page views
def track_page_view(page_id):
    r.incr(f"page:{page_id}:views")

# Get page views
def get_page_views(page_id):
    views = r.get(f"page:{page_id}:views")
    return int(views) if views else 0

# Usage
track_page_view("home")
track_page_view("home")
track_page_view("about")

print(get_page_views("home"))   # 2
print(get_page_views("about"))  # 1
```

### Example 3: Rate Limiting

```python
import redis
import time

r = redis.Redis()

def rate_limit(user_id, limit=100, window=3600):
    """Check if user is within rate limit"""
    key = f"ratelimit:{user_id}"
    
    # Increment counter
    current = r.incr(key)
    
    # Set expiration on first request
    if current == 1:
        r.expire(key, window)
    
    # Check if within limit
    return current <= limit

# Usage
user_id = "user:123"
for i in range(5):
    allowed = rate_limit(user_id, limit=100)
    print(f"Request {i+1}: {'Allowed' if allowed else 'Denied'}")
```

### Example 4: Cache with Expiration

```python
import redis
import json
import time

r = redis.Redis(decode_responses=True)

def get_user(user_id):
    cache_key = f"user:{user_id}"
    
    # Try cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Not in cache - fetch from database
    user = fetch_from_database(user_id)  # Your DB call
    
    # Store in cache for 1 hour
    r.setex(cache_key, 3600, json.dumps(user))
    
    return user

def update_user(user_id, data):
    # Update database
    save_to_database(user_id, data)
    
    # Invalidate cache
    r.delete(f"user:{user_id}")
```

## String Patterns

### Pattern 1: JSON Serialization

```python
import redis
import json

r = redis.Redis(decode_responses=True)

# Store complex object as JSON
user = {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'}
r.set('user:1', json.dumps(user))

# Retrieve and parse
stored = json.loads(r.get('user:1'))
print(stored['name'])  # Alice
```

### Pattern 2: Expiring Tokens

```python
import redis
import uuid

r = redis.Redis(decode_responses=True)

# Create and store token
token = str(uuid.uuid4())
user_id = 123
r.setex(f"token:{token}", 3600, user_id)  # Expires in 1 hour

# Later: Validate token
def validate_token(token):
    user_id = r.get(f"token:{token}")
    return user_id
```

### Pattern 3: Feature Flags

```python
import redis

r = redis.Redis(decode_responses=True)

# Store feature flag state
r.set("feature:new_ui", "true")
r.set("feature:beta_api", "false")

# Check flag
def is_enabled(feature_name):
    return r.get(f"feature:{feature_name}") == "true"

if is_enabled("new_ui"):
    # Use new UI
    pass
```

### Pattern 4: Distributed Lock

```python
import redis
import uuid
import time

r = redis.Redis(decode_responses=True)

def acquire_lock(resource, timeout=10):
    lock_id = str(uuid.uuid4())
    if r.set(f"lock:{resource}", lock_id, nx=True, ex=timeout):
        return lock_id
    return None

def release_lock(resource, lock_id):
    if r.get(f"lock:{resource}") == lock_id:
        r.delete(f"lock:{resource}")

# Usage
lock = acquire_lock("database", timeout=5)
try:
    # Do critical work
    pass
finally:
    release_lock("database", lock)
```

## Performance Characteristics

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| SET | O(1) | Simple write |
| GET | O(1) | Simple read |
| APPEND | O(N) | N = length of appended string |
| GETRANGE | O(N) | N = length of returned range |
| SETRANGE | O(N) | N = length of range |
| STRLEN | O(1) | Constant time |
| INCR/DECR | O(1) | Atomic operation |
| MGET | O(N) | N = number of keys |
| MSET | O(N) | N = number of key-value pairs |

### Memory Usage

```python
# String memory: key + value + overhead
# Each string has ~40 bytes overhead

# Examples:
SET key "hello"        # ~45 bytes (4 bytes key + 5 bytes value + overhead)
SET key "x" * 1000     # ~1040 bytes
SET key "x" * 1_000_000  # ~1MB + overhead
```

## Best Practices

### 1. Choose Appropriate Data Types

```python
# Good: Use strings for simple values
r.set('user:1:name', 'Alice')
r.set('count', 100)

# Consider: Use hashes for related data
r.hset('user:1', mapping={'name': 'Alice', 'email': 'alice@example.com'})
```

### 2. Set Expiration for Temporary Data

```python
# Good: Cache with expiration
r.setex('cache:results', 3600, json.dumps(results))

# Avoid: Permanent cache without invalidation
r.set('cache:results', json.dumps(results))
```

### 3. Use MGET/MSET for Multiple Keys

```python
# Good: Batch operations
r.mset({'key1': 'val1', 'key2': 'val2', 'key3': 'val3'})

# Avoid: Multiple round trips
r.set('key1', 'val1')
r.set('key2', 'val2')
r.set('key3', 'val3')
```

### 4. Use Atomic Operations

```python
# Good: Atomic increment
r.incr('counter')

# Avoid: Non-atomic read/modify/write
count = int(r.get('counter'))
r.set('counter', count + 1)  # Race condition!
```

### 5. Serialize Before Storing

```python
import json

# Good: Explicit serialization
data = {'key': 'value'}
r.set('data', json.dumps(data))

# Avoid: Implicit string conversion
r.set('data', data)  # Results in "{'key': 'value'}" (harder to parse)
```

## Common Mistakes

### ❌ Not Setting Expiration for Temporary Data

```python
# Bad: Cache grows forever
r.set('temp_data', data)

# Good: Set expiration
r.setex('temp_data', 300, data)  # 5 minutes
```

### ❌ Non-Atomic Increment

```python
# Bad: Race condition if multiple clients
count = int(r.get('count'))
r.set('count', count + 1)

# Good: Atomic
r.incr('count')
```

### ❌ Storing Large Objects Without Compression

```python
import json
import zlib

# Slow: Large JSON
r.set('data', json.dumps(large_object))

# Better: Compress
compressed = zlib.compress(json.dumps(large_object))
r.set('data', compressed)
```

### ❌ Assuming Numeric String Atomicity

```python
# Problematic
r.set('balance', '100')
balance = int(r.get('balance'))
r.set('balance', str(balance - 50))  # Non-atomic!

# Correct
r.decrby('balance', 50)  # Atomic
```

## Next Steps

- [Transactions](9-transaction.md) - Atomic multiple commands
- [Pipeline](8-pipeline.md) - Batch multiple commands
- [Data Structures](../2-data-structure/1-intro.md) - Other data types
- [Persistence](15-persistence.md) - How strings are stored

## Resources

- **String Documentation**: https://redis.io/docs/data-types/strings/
- **Commands**: https://redis.io/commands/?group=string
- **String Patterns**: https://redis.io/docs/design-patterns/

## Summary

- Strings are binary-safe sequences of bytes (up to 512MB)
- Atomic operations for counters (INCR/DECR)
- Efficient serialization with JSON
- Set expiration for temporary data (SETEX)
- Use MGET/MSET for batch operations
- Best for: Cache, sessions, counters, locks, temporary state
