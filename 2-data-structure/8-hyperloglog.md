# Redis HyperLogLog - Cardinality Estimation

## Overview

Redis HyperLogLog is a probabilistic data structure for estimating cardinality (unique count). Perfect for:
- **Unique Visitors**: Count unique users per day
- **Unique IPs**: Track distinct IPs accessing site
- **Deduplication**: Fast cardinality with minimal memory
- **Search Metrics**: Unique terms in search queries
- **Bloom Filters**: Approximate membership

### Why HyperLogLog?

- **Remarkable Memory**: ~12KB for billions of items
- **O(1) Operations**: Add and count in constant time
- **Probabilistic**: Small error rate (0.81% default)
- **Union Operations**: Merge HLLs atomically
- **Scalable**: No memory growth with data

---

## Core Commands

### Add & Count

```bash
# Add elements
PFADD users:today "user1" "user2" "user3"    # Returns: 1 (added)

# Check cardinality
PFCOUNT users:today                           # Returns: ~3

# Multiple HLLs
PFCOUNT users:today users:yesterday           # Union count
```

### Merge HLLs

```bash
# Merge multiple HLLs
PFMERGE users:week users:day1 users:day2 users:day3
# Result in users:week contains combined HLL
```

---

## Practical Examples

### Example 1: Daily Unique Visitors

```python
import redis
from datetime import datetime, timedelta

r = redis.Redis()

# Track visitors
def track_visitor(user_id):
    today = datetime.now().strftime('%Y-%m-%d')
    key = f'visitors:{today}'
    r.pfadd(key, user_id)
    r.expire(key, 7*86400)  # Keep for 7 days

# Get unique visitors
def get_visitors(date_str):
    key = f'visitors:{date_str}'
    return r.pfcount(key)

# Usage
track_visitor('user123')
track_visitor('user456')
track_visitor('user123')  # Duplicate ignored
print(f"Today: {get_visitors('2024-01-01')} visitors")
```

### Example 2: Unique IP Tracking

```python
# Track IPs per endpoint
def log_request(endpoint, ip_address):
    key = f'ips:endpoint:{endpoint}'
    r.pfadd(key, ip_address)

# Analytics
def get_unique_ips(endpoint):
    key = f'ips:endpoint:{endpoint}'
    return r.pfcount(key)

# Usage
log_request('/api/users', '192.168.1.1')
log_request('/api/users', '192.168.1.2')
log_request('/api/users', '192.168.1.1')  # Duplicate

print(f"Unique IPs: {get_unique_ips('/api/users')}")  # ~2
```

### Example 3: Weekly Uniques from Daily

```python
# Merge daily HLLs to get weekly
def get_week_visitors(year, week_num):
    keys = [f'visitors:{year}-W{week_num}-{day}' for day in range(1, 8)]
    
    # Merge all 7 days
    dest_key = f'visitors:week:{year}-W{week_num}'
    r.pfmerge(dest_key, *keys)
    
    # Get count
    count = r.pfcount(dest_key)
    r.expire(dest_key, 30*86400)  # Keep for 30 days
    
    return count
```

### Example 4: Search Analytics

```python
# Track unique search terms per user
def log_search(user_id, search_term):
    key = f'searches:user:{user_id}'
    r.pfadd(key, search_term)

# Get unique search terms
def get_search_variety(user_id):
    key = f'searches:user:{user_id}'
    return r.pfcount(key)

# Usage
log_search('user1', 'redis')
log_search('user1', 'redis')  # Duplicate
log_search('user1', 'database')
log_search('user1', 'cache')

print(f"User1 unique searches: {get_search_variety('user1')}")  # ~3
```

---

## Real-World Pattern: Visitor Funnel

```python
def track_funnel_event(funnel_name, stage, user_id):
    """Track user through funnel stages"""
    key = f'funnel:{funnel_name}:{stage}'
    r.pfadd(key, user_id)

def get_funnel_metrics(funnel_name):
    """Calculate funnel metrics"""
    stages = ['landing', 'signup', 'confirm', 'payment']
    counts = {}
    
    for stage in stages:
        key = f'funnel:{funnel_name}:{stage}'
        counts[stage] = r.pfcount(key)
    
    # Calculate dropoff
    for i in range(1, len(stages)):
        prev_stage = stages[i-1]
        curr_stage = stages[i]
        if counts[prev_stage] > 0:
            dropoff = (1 - counts[curr_stage] / counts[prev_stage]) * 100
            print(f"{prev_stage} → {curr_stage}: {dropoff:.1f}% drop")

# Usage
track_funnel_event('signup_flow', 'landing', 'user1')
track_funnel_event('signup_flow', 'signup', 'user1')
# user2 only reaches landing
track_funnel_event('signup_flow', 'landing', 'user2')

get_funnel_metrics('signup_flow')
```

---

## Performance

```
Operation              Time     Memory (items)
──────────────────────────────────────────────
PFADD                  O(1)     12KB for billions
PFCOUNT                O(1)     Constant
PFMERGE (N HLLs)      O(N)     12KB per HLL
Accuracy              0.81%     Error rate (tunable)
```

**Key Stat**: 1 million unique items = ~12KB memory!

---

## Accuracy & Error

Default error rate is 0.81%, which means:
- **1 million items**: ±8,100 error
- **100 million items**: ±810,000 error
- **1 billion items**: ±8.1 million error

Trade-off: Memory vs Accuracy

---

## Best Practices

### 1. Use HyperLogLog for Cardinality

```python
# ✅ DO: Count unique with HLL
r.pfadd('visitors:2024', 'user1', 'user2')
count = r.pfcount('visitors:2024')  # O(1), 12KB memory

# ❌ DON'T: Use set for billions of items
r.sadd('visitors:2024', 'user1', 'user2')
# For 1 billion items: ~40GB memory!
```

### 2. Merge Instead of Maintaining Total

```python
# ✅ DO: Merge daily HLLs for weekly
r.pfmerge('weekly:visitors', 'day1', 'day2', 'day3')
weekly = r.pfcount('weekly:visitors')

# ❌ DON'T: Manually add daily counts
# Doesn't account for duplicates across days!
total = daily_count1 + daily_count2 + daily_count3
```

### 3. Set Expiration on Old HLLs

```python
# ✅ DO: Expire old data
today = datetime.now().strftime('%Y-%m-%d')
key = f'visitors:{today}'
r.pfadd(key, user_id)
r.expire(key, 30*86400)  # Keep 30 days
```

### 4. Use with Caution for Critical Numbers

```python
# ✅ DO: Understand error rate
count = r.pfcount('visitors')  # ~0.81% error
# For reporting/analytics: OK
# For financial transactions: NO

# Use exact count for critical operations
exact_count = r.scard('critical:set')  # Exact but more memory
```

---

## Common Mistakes

### Mistake 1: Confusing HLL with Set

```python
# ❌ WRONG: Expecting exact deduplication
r.pfadd('users', 'alice')
r.pfadd('users', 'alice')
r.pfcount('users')  # Returns ~1 (not guaranteed)

# ✅ RIGHT: Use set for exact needs
r.sadd('users', 'alice')
r.sadd('users', 'alice')
r.scard('users')  # Returns exactly 1
```

### Mistake 2: Not Setting Expiration

```python
# ❌ WRONG: HLL grows with old data
r.pfadd('visitors:2020', users)
# 4 years later: still taking memory!

# ✅ RIGHT: Expire old HLLs
r.expire('visitors:2020', 365*86400)  # 1 year TTL
```

### Mistake 3: Wrong Merge Operation

```python
# ❌ WRONG: Adding counts (ignores duplicates across HLLs)
total = r.pfcount('day1') + r.pfcount('day2')
# This is wrong if users appear on both days

# ✅ RIGHT: Merge HLLs
r.pfmerge('total', 'day1', 'day2')
total = r.pfcount('total')  # Correct unique count
```

### Mistake 4: Using for Exact Small Numbers

```python
# ❌ WRONG: 10 visitors, 0.81% error = could be 9-11
r.pfadd('visitors', users)
count = r.pfcount('visitors')

# ✅ RIGHT: Use set for small numbers
r.sadd('visitors', users)
count = r.scard('visitors')  # Exact
```

---

## Advanced: Error Tuning

For specialized cases, you can use dense or sparse mode internally (automatic), but generally use default HLL.

---

## Next Steps

- **[Others (Bitmaps)](9-others.md)** - Bit-level operations
- **[Streams](6-stream.md)** - Event streams with unique tracking
- **[Sorted Sets](5-sorted-set.md)** - Cardinality by rank

