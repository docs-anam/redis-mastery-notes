# Redis Memory Eviction Policies

## Overview

When Redis reaches maxmemory limit, eviction policies determine which keys to delete. Choosing the right policy is critical for application behavior under memory pressure.

## Eviction Policies

### Available Policies

```
Eviction when maxmemory exceeded:

1. noeviction        - Error when full (don't delete anything)
2. allkeys-lru       - Delete least recently used (any key)
3. allkeys-lfu       - Delete least frequently used (any key)
4. allkeys-random    - Delete random key
5. volatile-lru      - Delete LRU key WITH TTL only
6. volatile-lfu      - Delete LFU key WITH TTL only
7. volatile-ttl      - Delete key with shortest TTL
8. volatile-random   - Delete random key WITH TTL only
```

### Policy Grouping

```
┌─ Keys with TTL only (volatile-*)
│  ├─ volatile-lru   : Least recently used
│  ├─ volatile-lfu   : Least frequently used
│  ├─ volatile-ttl   : Shortest expiration
│  └─ volatile-random: Random
│
├─ Any keys (allkeys-*)
│  ├─ allkeys-lru    : Least recently used
│  ├─ allkeys-lfu    : Least frequently used
│  └─ allkeys-random : Random
│
├─ Special
│  └─ noeviction     : No deletion (error)
```

## Configuration

### Setting Eviction Policy

```conf
# redis.conf
maxmemory 256mb              # Max memory allowed
maxmemory-policy allkeys-lru # Eviction policy

# Eviction samples (how many keys to check)
maxmemory-samples 5          # Check 5 keys (default)
                             # Higher = more accurate, slower
```

### Runtime Configuration

```python
import redis

r = redis.Redis()

# Check current policy
policy = r.config_get('maxmemory-policy')
print(f"Policy: {policy['maxmemory-policy']}")

# Change policy
r.config_set('maxmemory-policy', 'allkeys-lru')

# Check maxmemory
maxmem = r.config_get('maxmemory')
print(f"Max memory: {maxmem['maxmemory']} bytes")

# Set maxmemory
r.config_set('maxmemory', '256mb')
```

## Choosing the Right Policy

### Policy 1: allkeys-lru (Most Common)

**Best for:** General caching

```conf
maxmemory-policy allkeys-lru
```

**Logic:**
- Deletes least recently used keys
- Any key eligible (regardless of TTL)
- Good for general caches

**Use cases:**
- Web page cache
- API response cache
- User session fallback

```python
import redis

r = redis.Redis()

def cache_with_lru(key, value, ttl=3600):
    """Cache with LRU eviction"""
    r.setex(key, ttl, value)
    
    # If cache fills: LRU keys deleted
    # Recently used keys kept
```

### Policy 2: allkeys-lfu (Modern, Smarter)

**Best for:** Skewed access patterns

```conf
maxmemory-policy allkeys-lfu
```

**Logic:**
- Deletes least frequently used keys
- Tracks access frequency
- Smarter than LRU for many workloads

**Use cases:**
- Hotspot data (some keys much more popular)
- Product recommendations
- Analytics dashboards

```python
import redis

r = redis.Redis()

def cache_with_lfu(key, value):
    """Cache with LFU eviction"""
    r.set(key, value)
    
    # If popular: kept around
    # If unpopular: evicted first
```

### Policy 3: volatile-lru

**Best for:** Mixed TTL and non-TTL data

```conf
maxmemory-policy volatile-lru
```

**Logic:**
- Only deletes keys with TTL
- Keys without TTL are preserved
- Requires some keys to have TTL

**Use cases:**
- Cache + permanent config mix
- Session + config mix

```python
import redis

r = redis.Redis()

# Permanent config (no TTL)
r.set('config:version', '1.0')

# Cache (with TTL)
r.setex('cache:products', 3600, '[...]')

# When full: cache deleted, config kept
```

### Policy 4: volatile-ttl

**Best for:** Short-term data with varying TTL

```conf
maxmemory-policy volatile-ttl
```

**Logic:**
- Deletes keys with shortest remaining TTL
- Natural expiration optimization

**Use cases:**
- Time-sensitive notifications
- Temporary request data
- Session data

### Policy 5: noeviction (For Critical Data)

**Best for:** NO data loss acceptable

```conf
maxmemory-policy noeviction
requirepass "protect"
```

**Logic:**
- Returns error when full
- Forces application to handle full memory

**Use cases:**
- Critical transaction data
- Audit logs
- Financial records

```python
import redis

r = redis.Redis()

try:
    r.set('critical_data', value)
except redis.ResponseError:
    # Memory full, handle gracefully
    print("Redis memory full - application recovery needed")
    # Save to database
    # Alert ops team
```

## Eviction in Action

### Example: Eviction Triggered

```python
import redis

def demonstrate_eviction():
    """Show eviction in action"""
    
    # Small memory limit for demo
    r = redis.Redis()
    r.config_set('maxmemory', '100kb')
    r.config_set('maxmemory-policy', 'allkeys-lru')
    
    # Add keys
    for i in range(100):
        r.set(f'key{i}', 'x' * 1000)  # 1KB each
        print(f"Added key{i}, DB size: {r.dbsize()}")
        # After ~100 keys, oldest ones evicted
    
    print(f"Final keys: {r.dbsize()}")

# Run: demonstrate_eviction()
```

### Example: Access Patterns and LRU

```python
import redis
import time

def test_lru():
    """Test LRU eviction"""
    r = redis.Redis()
    r.flushdb()
    
    # Add keys with different access patterns
    for i in range(10):
        r.set(f'hot{i}', 'frequently accessed')
        r.set(f'cold{i}', 'rarely accessed')
    
    # Access hot keys frequently
    for _ in range(100):
        r.get('hot0')
        r.get('hot1')
        r.get('hot2')
    
    # Trigger eviction
    for i in range(50):
        r.set(f'new{i}', 'x' * 1000)  # Force eviction
    
    # Check what's left
    remaining = r.keys('*')
    hot_remaining = [k for k in remaining if b'hot' in k]
    cold_remaining = [k for k in remaining if b'cold' in k]
    
    print(f"Hot keys remaining: {len(hot_remaining)}")
    print(f"Cold keys remaining: {len(cold_remaining)}")
    # Result: More hot keys survive

test_lru()
```

## Monitoring Eviction

### Track Evicted Keys

```python
import redis
import time

def monitor_eviction():
    """Monitor key eviction"""
    r = redis.Redis()
    
    # Baseline
    info1 = r.info('stats')
    evicted_before = info1['evicted_keys']
    
    time.sleep(1)
    
    # After some activity
    info2 = r.info('stats')
    evicted_after = info2['evicted_keys']
    
    evicted_count = evicted_after - evicted_before
    print(f"Keys evicted: {evicted_count}")
    
    if evicted_count > 0:
        print("⚠️  Memory pressure causing evictions")
        print("Consider:")
        print("  - Increasing maxmemory")
        print("  - Changing eviction policy")
        print("  - Optimizing key sizes")

monitor_eviction()
```

### Memory Statistics

```python
import redis

def analyze_memory():
    """Analyze memory usage"""
    r = redis.Redis()
    
    memory = r.info('memory')
    stats = r.info('stats')
    
    used = memory['used_memory']
    max_mem = int(r.config_get('maxmemory')['maxmemory'])
    
    used_pct = (used / max_mem * 100) if max_mem > 0 else 0
    
    print(f"Memory used: {memory['used_memory_human']}")
    print(f"Max memory: {max_mem / (1024*1024):.0f}MB")
    print(f"Usage: {used_pct:.1f}%")
    print(f"Evicted keys: {stats['evicted_keys']}")
    print(f"Expired keys: {stats['expired_keys']}")
    
    # Status
    if used_pct > 90:
        print("⚠️  CRITICAL: Memory nearly full")
    elif used_pct > 80:
        print("⚠️  WARNING: High memory usage")
    else:
        print("✓ Healthy memory usage")

analyze_memory()
```

## Eviction Strategies

### Strategy 1: Aggressive Caching (Web Server)

```conf
# Caches are ephemeral, eviction OK
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 10  # More accurate

# With TTL
timeout 300
```

### Strategy 2: Session Storage

```conf
# Sessions have TTL, evict old sessions
maxmemory 1gb
maxmemory-policy volatile-lru
maxmemory-samples 5
```

### Strategy 3: Critical Data

```conf
# No eviction - application responsibility
maxmemory 512mb
maxmemory-policy noeviction  # Error on full
requirepass "strong_password"

# Enable persistence
appendonly yes
appendfsync always
```

### Strategy 4: Mixed Workload

```conf
# Hybrid approach
maxmemory 4gb
maxmemory-policy allkeys-lfu  # Intelligent eviction
aof-use-rdb-preamble yes
```

## Best Practices

### 1. Set Appropriate maxmemory

```python
# Good: 70-80% of available RAM
maxmemory = available_ram * 0.75

# Bad: Too high (swapping)
maxmemory = available_ram * 1.5

# Bad: Too low (constant eviction)
maxmemory = available_ram * 0.1
```

### 2. Match Policy to Workload

```
Caching:        allkeys-lru
Important data: noeviction or volatile-lru
Mixed:          allkeys-lfu
TTL-based:      volatile-ttl
```

### 3. Monitor Eviction

```python
def check_eviction_health():
    r = redis.Redis()
    info = r.info('stats')
    
    if info['evicted_keys'] > 100000:
        print("⚠️  High eviction - increase memory")
    
    memory = r.info('memory')
    frag = memory['mem_fragmentation_ratio']
    if frag > 1.5:
        print("⚠️  Memory fragmentation - restart Redis")

check_eviction_health()
```

### 4. Use TTL Strategically

```python
# Cache with TTL
r.setex('cache:key', 3600, 'value')  # 1 hour

# Persistent data without TTL
r.set('config:key', 'value')

# With volatile-lru: TTL'd caches evicted first
```

## Common Mistakes

### ❌ No maxmemory Set

```python
# Bad: Unbounded memory growth
# Redis uses all available RAM, then swaps

# Good: Set maxmemory
r.config_set('maxmemory', '256mb')
```

### ❌ Wrong Policy for Data

```python
# Bad: noeviction for ephemeral cache
maxmemory-policy noeviction
# Cache fills, returns errors

# Good: Match policy to data type
# Cache:         allkeys-lru
# Sessions:      volatile-lru
# Critical data: noeviction
```

### ❌ Ignoring Eviction

```python
# Bad: Not monitoring
# Don't know when eviction happens

# Good: Monitor eviction rate
if r.info('stats')['evicted_keys'] > 1000:
    alert("High eviction rate")
```

## Next Steps

- [Configuration](4-configuration.md) - Memory settings
- [Persistence](15-persistence.md) - Data durability
- [Monitoring](11-server-information.md) - Memory metrics

## Resources

- **Eviction Policies**: https://redis.io/docs/reference/eviction/
- **Memory Management**: https://redis.io/docs/management/optimization/
- **CONFIG**: https://redis.io/docs/manual/client-side-caching/

## Summary

- Eviction deletes keys when maxmemory exceeded
- Choose policy based on data type (cache vs critical)
- allkeys-lru: best for general caching
- allkeys-lfu: best for skewed access
- volatile-*: preserve non-TTL data
- noeviction: for critical data (application handles full)
- Monitor eviction to detect memory pressure
- Set appropriate maxmemory (70-80% of RAM)
