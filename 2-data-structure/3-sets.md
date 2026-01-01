# Redis Sets - Unique Collections & Membership

## Overview

Redis Sets are unordered collections of unique strings. Perfect for:
- **Tagging**: Article tags, user interests
- **Membership Testing**: User followers, group members
- **Set Operations**: Union, intersection, difference
- **Deduplication**: Unique visitor tracking
- **Recommendations**: Common interests between users

### Why Sets?

- **O(1) membership testing**: SISMEMBER in constant time
- **Set operations**: SINTER, SUNION, SDIFF (atomic)
- **No duplicates**: Automatically enforced
- **Memory efficient**: ~8 bytes per element
- **Fast operations**: All major ops are O(1) or O(N)

---

## Core Commands

### Add & Remove

```bash
# Add members
SADD users "alice" "bob" "charlie"    # Returns: 3

# Check membership
SISMEMBER users "alice"                # Returns: 1 (yes)
SISMEMBER users "david"                # Returns: 0 (no)

# Remove members
SREM users "bob"                       # Returns: 1 (removed)

# Remove random member
SPOP users                             # Returns: "charlie"
```

### Get Members

```bash
# Get all members
SMEMBERS users                         # Returns: all members

# Get count
SCARD users                            # Returns: number of members

# Get random members
SRANDMEMBER users                      # Returns: one random
SRANDMEMBER users 3                    # Returns: 3 random
```

### Set Operations

```bash
# Union of sets
SUNION set1 set2                       # All members from both

# Intersection
SINTER set1 set2                       # Common members only

# Difference
SDIFF set1 set2                        # In set1 but not set2

# Store results
SUNIONSTORE dest set1 set2
SINTERSTORE dest set1 set2
SDIFFSTORE dest set1 set2
```

---

## Practical Examples

### Example 1: Tag System

```python
import redis
r = redis.Redis()

# Tag an article
r.sadd('article:123:tags', 'python', 'redis', 'database')

# Check if tagged
if r.sismember('article:123:tags', 'python'):
    print("Article has python tag")

# Get all tags
tags = r.smembers('article:123:tags')
print(tags)  # {b'python', b'redis', b'database'}
```

### Example 2: Follower System

```python
# User A follows user B
r.sadd('user:b:followers', 'user:a')
r.sadd('user:a:following', 'user:b')

# Check if following
if r.sismember('user:a:following', 'user:b'):
    print("Following")

# Get follower count
count = r.scard('user:b:followers')
print(f"Followers: {count}")
```

### Example 3: Common Interests

```python
# User interests
r.sadd('interests:alice', 'python', 'redis', 'databases')
r.sadd('interests:bob', 'redis', 'databases', 'caching')

# Find common interests
common = r.sinter('interests:alice', 'interests:bob')
print(common)  # {b'redis', b'databases'}

# Recommend people with common interests
r.sinter('interests:alice', 'interests:bob', 'interests:charlie')
```

### Example 4: Unique Visitor Tracking

```python
# Track unique visitors per day
r.sadd('visitors:2024-01-01', 'user1', 'user2', 'user3')
r.sadd('visitors:2024-01-02', 'user2', 'user3', 'user4')

# Total unique visitors
r.scard('visitors:2024-01-01')  # 3
r.scard('visitors:2024-01-02')  # 3

# Visitors on both days
r.sinter('visitors:2024-01-01', 'visitors:2024-01-02')
# {b'user2', b'user3'}
```

---

## Real-World Patterns

### Pattern 1: Social Network

```python
# User A follows user B
def follow_user(user_a, user_b):
    r.sadd(f'user:{user_a}:following', user_b)
    r.sadd(f'user:{user_b}:followers', user_a)

# Get mutual friends
def get_mutual_friends(user_a, user_b):
    a_following = f'user:{user_a}:following'
    b_followers = f'user:{user_b}:followers'
    return r.sinter(a_following, b_followers)

# Get suggestions (friends of friends)
def get_suggestions(user_a):
    following = f'user:{user_a}:following'
    # Get all friends of my friends
    friends_of_friends = set()
    for friend in r.smembers(following):
        friends = r.smembers(f'user:{friend}:following')
        friends_of_friends.update(friends)
    
    # Remove already following
    friends_of_friends -= r.smembers(following)
    return friends_of_friends
```

### Pattern 2: Online Users Tracking

```python
def mark_online(user_id):
    """Mark user as online"""
    r.sadd('users:online', user_id)

def mark_offline(user_id):
    """Mark user as offline"""
    r.srem('users:online', user_id)

def get_online_count():
    """Get total online users"""
    return r.scard('users:online')

def are_friends_online(user_id):
    """Check which friends are online"""
    friends = r.smembers(f'user:{user_id}:friends')
    online = r.smembers('users:online')
    return friends & online  # Intersection
```

### Pattern 3: Content Recommendations

```python
def get_recommendations(user_id, num_recommendations=5):
    """Get recommended content based on interests"""
    # Get user's interests
    interests = r.smembers(f'user:{user_id}:interests')
    
    # Get popular content for each interest
    recommended = set()
    for interest in interests:
        content = r.smembers(f'content:interest:{interest}')
        recommended.update(content)
    
    # Remove already viewed
    viewed = r.smembers(f'user:{user_id}:viewed')
    recommended -= viewed
    
    return list(recommended)[:num_recommendations]
```

---

## Performance

```
Operation              Time     Memory
──────────────────────────────────────
SADD                   O(1)     8 bytes per element
SREM                   O(1)     Constant
SISMEMBER              O(1)     Constant
SMEMBERS               O(N)     Returns N elements
SINTER                 O(N×M)   Where N=smallest set
SUNION                 O(N)     All members
SDIFF                  O(N)     All members
SPOP (single)          O(1)     Constant
```

---

## Best Practices

### 1. Use Sets for Membership Testing

```python
# ✅ DO: Use sets
r.sadd('article:tags', 'python')
if r.sismember('article:tags', 'python'):
    pass

# ❌ DON'T: Use lists or strings
tags_list = ['python', 'redis']  # O(N) lookup
if 'python' in tags_list:
    pass
```

### 2. Use Set Operations for Queries

```python
# ✅ DO: Let Redis handle intersection
common = r.sinter('interests:alice', 'interests:bob')

# ❌ DON'T: Fetch and compute in application
a_interests = set(r.smembers('interests:alice'))
b_interests = set(r.smembers('interests:bob'))
common = a_interests & b_interests
```

### 3. Store Results with XXSTORE Commands

```python
# ✅ DO: Atomic operation
r.sinterstore('common:interests', 'user:a:interests', 'user:b:interests')

# ❌ DON'T: Separate operations
common = r.sinter(...)
r.delete('common:interests')
# Gap between operations!
```

### 4. Monitor Set Size

```python
# ✅ DO: Track set growth
count = r.scard('tags')
if count > 10000:
    alert("Too many tags!")
```

---

## Common Mistakes

### Mistake 1: Using Lists Instead of Sets

```python
# ❌ WRONG: O(N) membership testing
r.rpush('tags', 'python', 'redis')
if 'python' in r.lrange('tags', 0, -1):  # O(N)
    pass

# ✅ RIGHT: O(1) with sets
r.sadd('tags', 'python', 'redis')
if r.sismember('tags', 'python'):  # O(1)
    pass
```

### Mistake 2: Not Deduplicating Input

```python
# ❌ WRONG: Duplicates in input
r.sadd('tags', 'python', 'python', 'redis')
# Sets handle this, but wasteful

# ✅ RIGHT: Deduplicate first
tags = set(['python', 'redis'])
r.sadd('tags', *tags)
```

### Mistake 3: Ignoring Set Operation Cost

```python
# ❌ WRONG: Large set operations are slow
# SINTER on million-element sets = slow

# ✅ RIGHT: Precompute if used frequently
r.sinterstore('common:users', 'set1', 'set2')
r.expire('common:users', 3600)  # Cache for 1 hour
```

### Mistake 4: Wrong Data Structure Choice

```python
# ❌ WRONG: Using set when order matters
r.sadd('leaderboard', user1, user2)
# Sets don't preserve order or scores

# ✅ RIGHT: Use sorted set for rankings
r.zadd('leaderboard', {'user1': 100, 'user2': 95})
```

---

## Next Steps

- **[Hashes](4-hashes.md)** - Structured objects
- **[Sorted Sets](5-sorted-set.md)** - Rankings
- **[Streams](6-stream.md)** - Event logs

