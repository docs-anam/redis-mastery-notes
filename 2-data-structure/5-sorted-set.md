# Redis Sorted Sets - Rankings & Leaderboards

## Overview

Sorted Sets are collections of unique members with associated scores, ordered by score. Perfect for:
- **Leaderboards**: Game rankings, scores
- **Priority Queues**: Tasks with priorities
- **Time-based Sorting**: Recent posts, trending topics
- **Rate Limiting**: Request rates per user
- **Metrics**: Percentiles, histogram data

### Why Sorted Sets?

- **O(log N) operations**: ZADD, ZREM, ZSCORE all logarithmic
- **Ordered by score**: Automatic sorting
- **Range queries**: Get top N items efficiently
- **Atomic operations**: Update score and position together
- **Rich queries**: Rank, range, score operations

---

## Core Commands

### Add & Update

```bash
# Add members with scores
ZADD leaderboard 100 "alice" 95 "bob" 110 "charlie"

# Update score (same syntax)
ZADD leaderboard 105 "alice"

# Add only if doesn't exist
ZADD leaderboard NX 110 "david"
```

### Get & Query

```bash
# Get score of member
ZSCORE leaderboard "alice"                  # 105

# Get rank (0-indexed, ascending)
ZRANK leaderboard "alice"                   # Position from lowest

# Get rank (descending - for leaderboards)
ZREVRANK leaderboard "alice"                # Position from highest

# Get count
ZCARD leaderboard                           # Total members
```

### Range Operations

```bash
# Get range by rank
ZRANGE leaderboard 0 2                      # Top 3 (lowest)
ZREVRANGE leaderboard 0 2                   # Top 3 (highest)

# Get with scores
ZREVRANGE leaderboard 0 9 WITHSCORES        # Top 10 with scores

# Get by score range
ZRANGEBYSCORE leaderboard 100 110           # All with score 100-110
ZREVRANGEBYSCORE leaderboard 110 100        # Same, descending
```

### Increment

```bash
# Increment score
ZINCRBY leaderboard 5 "alice"               # Add 5 to alice's score
```

### Remove

```bash
# Remove member
ZREM leaderboard "bob"

# Remove by rank
ZREMRANGEBYRANK leaderboard 0 2             # Remove bottom 3

# Remove by score
ZREMRANGEBYSCORE leaderboard 0 50           # Remove score 0-50
```

---

## Practical Examples

### Example 1: Game Leaderboard

```python
import redis
r = redis.Redis()

# Update scores
r.zadd('game:leaderboard', {'alice': 1000, 'bob': 950})

# New score
r.zincrby('game:leaderboard', 50, 'alice')  # alice now has 1050

# Get top 10
top_10 = r.zrevrange('game:leaderboard', 0, 9, withscores=True)
for player, score in top_10:
    print(f"{player}: {score}")

# Get player's rank
rank = r.zrevrank('game:leaderboard', 'alice')
score = r.zscore('game:leaderboard', 'alice')
print(f"Rank: {rank}, Score: {score}")
```

### Example 2: Recent Posts

```python
import time

# Add post (score = timestamp)
now = time.time()
r.zadd('posts:recent', {'post:123': now})
r.zadd('posts:recent', {'post:456': now + 1})

# Get 10 most recent
recent = r.zrevrange('posts:recent', 0, 9)
for post_id in recent:
    post = r.hgetall(post_id)
    print(post)
```

### Example 3: Priority Queue

```python
# Add tasks with priority
r.zadd('task:queue', {
    'send_email:1': 10,  # Priority 10
    'process_payment:1': 5,  # Priority 5
    'notify:1': 20  # Priority 20
})

# Get highest priority task
task = r.zrevrange('task:queue', 0, 0)
if task:
    r.zrem('task:queue', task[0])
    process(task[0])
```

### Example 4: Rate Limiting by Score

```python
import time

def check_rate_limit(user_id, max_requests=100, window=60):
    """Check if user exceeded limit"""
    key = f'ratelimit:{user_id}'
    now = time.time()
    window_start = now - window
    
    # Remove old entries
    r.zremrangebyscore(key, 0, window_start)
    
    # Count recent
    count = r.zcard(key)
    
    if count < max_requests:
        r.zadd(key, {str(now): now})
        r.expire(key, window + 1)
        return True
    return False
```

---

## Real-World Patterns

### Pattern 1: Trending Topics

```python
def increment_topic_score(topic, count=1):
    """Increment topic score (popularity)"""
    r.zincrby('trending:topics', count, topic)

def get_trending(limit=10):
    """Get trending topics"""
    return r.zrevrange('trending:topics', 0, limit-1, withscores=True)

# Usage
increment_topic_score('python')  # Someone mentioned Python
trending = get_trending()
```

### Pattern 2: Percentile Ranking

```python
def add_latency_sample(service, latency_ms):
    """Record latency measurement"""
    r.zadd(f'latency:{service}', {str(time.time()): latency_ms})

def get_percentile(service, percentile=95):
    """Get P95 latency"""
    samples = r.zrange(f'latency:{service}', 0, -1)
    if not samples:
        return None
    
    # Calculate percentile
    index = int(len(samples) * (percentile / 100))
    return float(samples[index])
```

---

## Performance

```
Operation                    Time         Memory
─────────────────────────────────────────────
ZADD                        O(log N)      16 bytes per member
ZREM                        O(log N)      Constant
ZSCORE                      O(1)          Constant
ZRANK/ZREVRANK              O(log N)      Constant
ZRANGE (N elements)         O(log N + N)  Returns N elements
ZREVRANGE                   O(log N + N)  Returns N elements
ZRANGEBYSCORE               O(log N + N)  Variable
ZINCRBY                     O(log N)      Constant
```

**Key Insight**: Range queries return N elements, so ZREVRANGE leaderboard 0 9 returns 10 items O(log N + 10).

---

## Best Practices

### 1. Use Sorted Sets for Rankings

```python
# ✅ DO: Sorted set for leaderboard
r.zadd('leaderboard', {'alice': 100, 'bob': 95})
top = r.zrevrange('leaderboard', 0, 9)

# ❌ DON'T: Hash and manual sorting
r.hset('scores', mapping={'alice': 100, 'bob': 95})
# Then fetch all and sort in app (O(N log N))
```

### 2. Use ZREVRANGE for Top Results

```python
# ✅ DO: Get top N directly
top_10 = r.zrevrange('leaderboard', 0, 9, withscores=True)

# ❌ DON'T: Get all and slice
all_scores = r.zrange('leaderboard', 0, -1)
top_10 = all_scores[-10:]  # Wrong order!
```

### 3. Atomic Score Updates

```python
# ✅ DO: Atomic increment
r.zincrby('leaderboard', 10, 'alice')

# ❌ DON'T: Read-modify-write
score = float(r.zscore('leaderboard', 'alice') or 0)
r.zadd('leaderboard', {'alice': score + 10})
# Race condition between read and write!
```

### 4. Use ZADD with Options

```python
# ✅ DO: Use CH option to track changes
result = r.zadd('leaderboard', {'alice': 100, 'bob': 95}, ch=True)
# Returns: number of changed elements

# ✅ DO: Use NX for conditional add
r.zadd('unique:ids', {'id1': 1}, nx=True)  # Only if doesn't exist
```

---

## Common Mistakes

### Mistake 1: Wrong Range Boundaries

```python
# ❌ WRONG: Inclusive range
r.zrevrange('leaderboard', 0, 9)   # Gets 10 items (0-9 inclusive)

# ❌ WRONG: Python slice semantics
# redis: 0 to 9 = 10 items
# python: [0:9] = 9 items (not including 9)
```

### Mistake 2: Confusing ZRANK and ZREVRANK

```python
# ❌ WRONG: Using wrong rank function
rank = r.zrank('leaderboard', 'alice')          # Rank from bottom
# But you want rank from top!

# ✅ RIGHT: Use ZREVRANK for leaderboards
rank = r.zrevrank('leaderboard', 'alice')       # 0 = top player
```

### Mistake 3: Not Removing Old Entries

```python
# ❌ WRONG: Scores accumulate forever
r.zadd('trending', {topic: timestamp})
# After years: millions of old topics!

# ✅ RIGHT: Remove old entries
r.zremrangebyscore('trending', 0, time.time() - 86400)  # Remove older than 24h
```

### Mistake 4: Score Collisions

```python
# ❌ WRONG: Same score, unpredictable order
r.zadd('leaderboard', {'alice': 100, 'bob': 100})
# Which one is rank 1?

# ✅ RIGHT: Include tiebreaker in score
# Or use secondary sort in application
sorted_players = sorted(r.zrevrange(...), key=lambda x: (score, name))
```

---

## Next Steps

- **[Streams](6-stream.md)** - Event logs
- **[Hashes](4-hashes.md)** - Objects
- **[Lists](2-lists.md)** - Queues

