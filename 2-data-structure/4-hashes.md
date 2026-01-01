# Redis Hashes - Objects & Structured Data

## Overview

Redis Hashes are maps of field → value pairs. Perfect for:
- **User Profiles**: Name, email, age, location
- **Configuration Objects**: Settings, parameters
- **Document-like Data**: Nested structures
- **Database Rows**: Alternative to relational queries
- **Counters**: Per-field increment operations

### Why Hashes?

- **O(1) field access**: Direct lookups by field
- **Atomic field updates**: Update single field without touching others
- **Memory efficient**: Overhead split across fields
- **Intuitive mapping**: Like Python dicts or JSON objects
- **No serialization**: Work with fields directly

---

## Core Commands

### Set & Get

```bash
# Set single field
HSET user:123 name "Alice" age "30"      # Returns: 2 (fields added)

# Get field
HGET user:123 name                        # Returns: "Alice"

# Get multiple fields
HMGET user:123 name age city              # Returns: ["Alice", "30", nil]

# Get all fields and values
HGETALL user:123                          # Returns: all fields and values
```

### Check & Count

```bash
# Check if field exists
HEXISTS user:123 name                     # Returns: 1 (yes)

# Get field count
HLEN user:123                             # Returns: number of fields

# Get all field names
HKEYS user:123                            # Returns: [name, age, ...]

# Get all values
HVALS user:123                            # Returns: [Alice, 30, ...]
```

### Increment

```bash
# Increment integer field
HINCRBY user:123 age 1                    # Returns: 31

# Increment float field
HINCRBYFLOAT user:123 balance 10.50       # Returns: 110.50
```

### Delete & Update

```bash
# Delete field
HDEL user:123 age                         # Returns: 1 (deleted)

# Update field (same as HSET)
HSET user:123 age 31                      # Returns: 0 (updated)

# Set only if doesn't exist
HSETNX user:123 email "alice@example.com" # Returns: 1 (set)
```

---

## Practical Examples

### Example 1: User Profile

```python
import redis
r = redis.Redis()

# Create profile
profile_data = {
    'name': 'Alice',
    'email': 'alice@example.com',
    'age': '30',
    'city': 'New York'
}
r.hset('user:123', mapping=profile_data)

# Get single field
name = r.hget('user:123', 'name')         # b'Alice'

# Get all
all_data = r.hgetall('user:123')

# Update one field
r.hset('user:123', 'age', '31')
```

### Example 2: Product Catalog

```python
# Store product
product_data = {
    'name': 'Redis Guide',
    'price': '29.99',
    'stock': '100',
    'rating': '4.8'
}
r.hset('product:456', mapping=product_data)

# Decrement stock
r.hincrby('product:456', 'stock', -1)

# Get price
price = r.hget('product:456', 'price')
```

### Example 3: Session Storage

```python
# Store session
session_data = {
    'user_id': '123',
    'username': 'alice',
    'logged_in_at': '2024-01-01T10:00:00',
    'ip_address': '192.168.1.1'
}
r.hset('session:abc123', mapping=session_data)
r.expire('session:abc123', 3600)  # 1 hour expiration

# Retrieve session
session = r.hgetall('session:abc123')
```

### Example 4: Analytics Counter

```python
# Track events per day
r.hincrby('analytics:2024-01-01', 'page_views', 1)
r.hincrby('analytics:2024-01-01', 'visits', 1)
r.hincrby('analytics:2024-01-01', 'signups', 1)

# Get all metrics
metrics = r.hgetall('analytics:2024-01-01')
```

---

## Real-World Patterns

### Pattern 1: User Profile with Expiration

```python
def create_user_profile(user_id, profile_data, ttl=86400):
    """Create profile with 24-hour expiration"""
    key = f'user:{user_id}:profile'
    r.hset(key, mapping=profile_data)
    r.expire(key, ttl)
    return key

def get_user_profile(user_id):
    """Retrieve user profile"""
    key = f'user:{user_id}:profile'
    profile = r.hgetall(key)
    return profile if profile else None
```

### Pattern 2: Configuration Storage

```python
# Store app configuration
config = {
    'max_connections': '100',
    'timeout': '30',
    'debug': 'false',
    'version': '1.0.0'
}
r.hset('config:app', mapping=config)

# Get config value
max_conn = r.hget('config:app', 'max_connections')

# Update one setting
r.hset('config:app', 'debug', 'true')
```

### Pattern 3: Leaderboard Scores

```python
def update_score(game_id, player_id, score):
    """Update player score"""
    r.hset(f'game:{game_id}:scores', player_id, score)

def get_leaderboard(game_id):
    """Get all scores"""
    scores = r.hgetall(f'game:{game_id}:scores')
    # Sort by score (in application)
    return sorted(scores.items(), key=lambda x: -int(x[1]))
```

---

## Performance

```
Operation              Time     Memory
──────────────────────────────────────
HSET                   O(1)     16 bytes per field
HGET                   O(1)     Constant
HGETALL                O(N)     Returns N fields
HLEN                   O(1)     Constant
HINCRBY                O(1)     Constant
HDEL                   O(N)     For N fields
HMGET                  O(N)     For N fields
```

---

## Best Practices

### 1. Use Hashes for Objects

```python
# ✅ DO: Structured data with hash
r.hset('user:123', mapping={
    'name': 'Alice',
    'age': '30',
    'email': 'alice@example.com'
})

# ❌ DON'T: Serialize to JSON
import json
data = json.dumps({'name': 'Alice', 'age': 30})
r.set('user:123', data)
# Now updating age requires full parse/serialize
```

### 2. Atomic Field Updates

```python
# ✅ DO: Update single field
r.hset('user:123', 'age', '31')

# ❌ DON'T: Get-modify-set (race condition)
user = r.hgetall('user:123')
user['age'] = 31
r.delete('user:123')
r.hset('user:123', mapping=user)
```

### 3. Use Increment for Counters

```python
# ✅ DO: Atomic increment
r.hincrby('user:123', 'post_count', 1)

# ❌ DON'T: Manual increment
count = int(r.hget('user:123', 'post_count') or 0)
r.hset('user:123', 'post_count', count + 1)
```

### 4. Expire Entire Hash

```python
# ✅ DO: Set TTL on entire hash
r.hset('session:abc', mapping=session_data)
r.expire('session:abc', 3600)
```

---

## Common Mistakes

### Mistake 1: Storing Complex Nested Data

```python
# ❌ WRONG: Nested object in field
user = {
    'profile': {
        'name': 'Alice',
        'age': 30,
        'address': {
            'city': 'NYC',
            'zip': '10001'
        }
    }
}
r.hset('user:123', 'profile', json.dumps(user['profile']))
# Now updating nested field is hard!

# ✅ RIGHT: Flatten the structure
r.hset('user:123', mapping={
    'name': 'Alice',
    'age': '30',
    'city': 'NYC',
    'zip': '10001'
})
```

### Mistake 2: Not Setting Expiration

```python
# ❌ WRONG: Session never expires
r.hset('session:123', mapping=session_data)
# After years: millions of old sessions!

# ✅ RIGHT: Always set TTL
r.hset('session:123', mapping=session_data)
r.expire('session:123', 3600)
```

### Mistake 3: Race Conditions

```python
# ❌ WRONG: Non-atomic read-modify-write
score = int(r.hget('game:player:123', 'score') or 0)
r.hset('game:player:123', 'score', score + 10)
# Between HGET and HSET, another update could occur

# ✅ RIGHT: Use atomic operation
r.hincrby('game:player:123', 'score', 10)
```

### Mistake 4: Large Hash Values

```python
# ❌ WRONG: Storing large text in field
large_text = "..." * 1000000  # 1MB
r.hset('doc:123', 'content', large_text)

# ✅ RIGHT: Store separately as string
r.set('doc:123:content', large_text)
r.hset('doc:123', mapping={
    'title': 'Title',
    'author': 'Author',
    'size': len(large_text)
})
```

---

## Next Steps

- **[Sorted Sets](5-sorted-set.md)** - Rankings and scoring
- **[Streams](6-stream.md)** - Event logs
- **[Lists](2-lists.md)** - Queues and sequences

