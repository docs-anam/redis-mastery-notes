# Redis Sets

## Overview
A Redis Set is an unordered collection of unique strings. Sets are optimized for membership testing and set operations, making them perfect for tasks requiring uniqueness and fast lookups.

## Key Characteristics
- **Unordered**: No guaranteed order of elements
- **Unique**: Each element appears only once (automatic deduplication)
- **Fast lookups**: O(1) average time complexity for membership checks
- **Set Operations**: Efficient intersection, union, and difference operations
- **Memory Efficient**: Optimized internal hash table implementation
- **No duplicates**: Attempting to add existing member silently succeeds with no effect

## Common Commands

### Basic Operations
| Command | Complexity | Description |
|---------|-----------|-------------|
| `SADD key member [member ...]` | O(N) | Add one or more members |
| `SREM key member [member ...]` | O(N) | Remove members |
| `SMEMBERS key` | O(N) | Get all members |
| `SCARD key` | O(1) | Get set cardinality (count) |
| `SISMEMBER key member` | O(1) | Check if member exists |
| `SRANDMEMBER key [count]` | O(N) | Get random members |
| `SPOP key [count]` | O(N) | Remove and return random members |

### Set Operations
| Command | Complexity | Description |
|---------|-----------|-------------|
| `SINTER key [key ...]` | O(N*M) | Get intersection of sets |
| `SINTERSTORE dest key [key ...]` | O(N*M) | Store intersection result |
| `SUNION key [key ...]` | O(N) | Get union of sets |
| `SUNIONSTORE dest key [key ...]` | O(N) | Store union result |
| `SDIFF key [key ...]` | O(N) | Get difference of sets |
| `SDIFFSTORE dest key [key ...]` | O(N) | Store difference result |
| `SMOVE source dest member` | O(1) | Move member between sets |

## Use Cases

### 1. Unique Visitor/User Tracking
Track unique IPs, user IDs, or visitors with daily/hourly granularity.

```redis
// Track daily unique visitors
SADD visitors:2024-01-15 "192.168.1.1"
SADD visitors:2024-01-15 "192.168.1.2"
SADD visitors:2024-01-15 "192.168.1.1"  // No effect, already exists

// Get total unique visitors
SCARD visitors:2024-01-15  // Returns: 2

// Check if specific user visited
SISMEMBER visitors:2024-01-15 "192.168.1.1"  // Returns: 1 (true)

// Compare visitors between days
SINTER visitors:2024-01-15 visitors:2024-01-16  // Users visiting both days
```

**Real-world scenario**: Analytics dashboard, daily unique visitors report.

### 2. Tags & Categories
Store and manage tags efficiently for content.

```redis
// Add tags to article
SADD article:123:tags "redis" "database" "cache" "in-memory"

// Get all tags for article
SMEMBERS article:123:tags

// Check if article has specific tag
SISMEMBER article:123:tags "redis"  // Returns: 1

// Find articles with tag
SADD tag:redis:articles "article:123" "article:456" "article:789"

// Remove tag from article
SREM article:123:tags "in-memory"
```

**Real-world scenario**: Blog categorization, product attributes, content management.

### 3. Friends/Followers Network
Manage social relationships efficiently.

```redis
// User 1 follows users
SADD user:1:following "user:2" "user:3" "user:4"
SADD user:1:followers "user:5" "user:6"

// User 2 follows users
SADD user:2:following "user:1" "user:3" "user:7"
SADD user:2:followers "user:1" "user:8"

// Check if user 1 follows user 2
SISMEMBER user:1:following "user:2"  // Returns: 1

// Find mutual followers
SINTER user:1:followers user:2:followers  // People following both

// Find mutual following
SINTER user:1:following user:2:following  // Users both follow
```

**Real-world scenario**: Twitter followers, Facebook friends, LinkedIn connections.

### 4. Common Interests & Recommendations
Find users with overlapping interests for recommendations.

```redis
// Store interests
SADD user:1:interests "coding" "gaming" "music" "travel"
SADD user:2:interests "coding" "sports" "music" "photography"
SADD user:3:interests "coding" "gaming" "music" "design"

// Find common interests between users
SINTER user:1:interests user:2:interests  // Returns: "coding", "music"
SINTER user:1:interests user:3:interests  // Returns: "coding", "gaming", "music"

// Find unique interests
SDIFF user:1:interests user:2:interests  // Returns: "gaming", "travel"
```

**Real-world scenario**: Recommendation systems, user matching, interest-based discovery.

### 5. Deduplication & Uniqueness
Automatic deduplication of data.

```redis
// Store emails
SADD emails:confirmed "john@example.com" "jane@example.com" "bob@example.com"
SADD emails:pending "jane@example.com" "alice@example.com"

// Get unconfirmed emails
SDIFF emails:pending emails:confirmed  // Returns: "alice@example.com"

// Move email from pending to confirmed
SMOVE emails:pending emails:confirmed "jane@example.com"
```

**Real-world scenario**: Email lists, user registrations, data cleaning.

### 6. IP Whitelist/Blacklist
Manage access control lists.

```redis
// IP whitelist
SADD firewall:whitelist "192.168.1.1" "192.168.1.2" "10.0.0.1"
SADD firewall:blacklist "203.0.113.1" "198.51.100.5"

// Check if IP is allowed
SISMEMBER firewall:whitelist "192.168.1.1"  // Returns: 1
SISMEMBER firewall:blacklist "203.0.113.1"  // Returns: 1

// Find IPs in both lists
SINTER firewall:whitelist firewall:blacklist  // Should be empty
```

### 7. Random Selection (Sampling)
Pick random items for surveys, notifications, etc.

```redis
// Store user IDs
SADD active:users "user:1" "user:2" "user:3" "user:4" "user:5"

// Get 3 random users for survey
SRANDMEMBER active:users 3  // Returns 3 random users

// Remove and return random user
SPOP online:users  // Returns 1 random user and removes it
SPOP online:users 5  // Returns 5 random users and removes them
```

**Real-world scenario**: A/B testing, random user sampling, lottery selection.

## Advanced Patterns

### Bloom Filter Simulation
```redis
// Check if URL already crawled
SADD crawled:urls "https://example.com"
SISMEMBER crawled:urls "https://example.com"  // Returns: 1
```

### Multi-set Intersection
```redis
// Find users active in multiple segments
SINTER segment:premium segment:usa segment:active

// Store result
SINTERSTORE result:target_users segment:premium segment:usa segment:active
SCARD result:target_users  // Get count of target users
```

### Set Union for Analytics
```redis
// Total unique users across all days
SUNION visitors:2024-01-15 visitors:2024-01-16 visitors:2024-01-17

// Store for reporting
SUNIONSTORE visitors:week:total visitors:2024-01-15 visitors:2024-01-16
```

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Add/Remove | O(1) | Average case, per member |
| Membership check | O(1) | Average case |
| Get all members | O(N) | N = cardinality |
| Intersection | O(N*M) | N = smallest set, M = number of sets |
| Union | O(N) | N = total elements |
| Difference | O(N) | N = cardinality of first set |
| Random element | O(1) | Expected, with small cardinality advantage |

## Memory Considerations

- Sets use hash table internally for O(1) membership
- Memory overhead per element: ~8 bytes pointer + value size
- Smaller sets more memory-efficient than equivalent lists
- String interning optimization for duplicate values across sets

## Limitations

- **Unordered**: Cannot retrieve elements by rank
- **No duplication**: Cannot store same element twice (use Lists or Sorted Sets)
- **No scoring**: Elements have no associated values (use Hashes or Sorted Sets)
- **Set operations complexity**: O(N*M) intersections can be expensive

## Best Practices

1. **Use SISMEMBER for membership tests**: O(1) is much faster than SMEMBERS
2. **Cache intersection results**: Use SINTERSTORE for frequent operations
3. **Limit set sizes**: Very large sets increase memory usage and operation time
4. **Index by tag**: Pre-calculate reverse indexes (tag → items)
5. **Expire sets**: Use EXPIRE for temporary sets like sessions or rate limits

## Common Pitfalls

1. **SMEMBERS for large sets**: O(N) operation can block Redis
2. **Expensive intersections**: Multiple large sets intersection is slow
3. **Memory growth**: Sets without EXPIRE can consume unbounded memory
4. **Type mismatches**: Adding non-string values requires serialization
