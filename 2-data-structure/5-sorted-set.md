# Redis Sorted Sets

## Overview
Sorted Sets (ZSets) combine properties of Sets and Hashes. Each unique member is associated with a floating-point score used for ordering. This structure enables efficient ranking, leaderboards, and priority-based queries.

## Key Characteristics
- **Unique Members**: No duplicate elements (like Sets)
- **Scored**: Each member has a floating-point score (-inf to +inf)
- **Ordered**: Members automatically sorted by score (low to high)
- **Efficient Ranking**: O(log N) for most operations
- **Range Queries**: Efficient score-based and index-based ranges
- **Skip List Implementation**: Optimized for both random access and range operations

## Common Commands

### Adding & Removing
| Command | Complexity | Description |
|---------|-----------|-------------|
| `ZADD key score member [score member ...]` | O(N log M) | Add members with scores |
| `ZREM key member [member ...]` | O(N log M) | Remove members |
| `ZPOPMIN key [count]` | O(N log M) | Remove lowest scored members |
| `ZPOPMAX key [count]` | O(N log M) | Remove highest scored members |
| `BZPOPMIN key timeout` | O(N log M) | Blocking pop min |
| `BZPOPMAX key timeout` | O(N log M) | Blocking pop max |

### Querying by Index
| Command | Complexity | Description |
|---------|-----------|-------------|
| `ZRANGE key start stop [WITHSCORES]` | O(N) | Get members by index (low to high) |
| `ZREVRANGE key start stop [WITHSCORES]` | O(N) | Get members by index (high to low) |
| `ZCARD key` | O(1) | Count total members |
| `ZRANK key member` | O(log M) | Get rank (0-indexed, low to high) |
| `ZREVRANK key member` | O(log M) | Get rank (high to low) |

### Querying by Score
| Command | Complexity | Description |
|---------|-----------|-------------|
| `ZRANGEBYSCORE key min max [LIMIT]` | O(N log M) | Get members in score range |
| `ZREVRANGEBYSCORE key max min [LIMIT]` | O(N log M) | Reverse score range |
| `ZCOUNT key min max` | O(log M) | Count members in score range |
| `ZSCORE key member` | O(1) | Get member's score |
| `ZMSCORE key member [member ...]` | O(N) | Get multiple scores |

### Score Operations
| Command | Complexity | Description |
|---------|-----------|-------------|
| `ZINCRBY key increment member` | O(log M) | Increment member's score |
| `ZINTERSTORE dest [weights]` | O(N*K log M) | Store intersection |
| `ZUNIONSTORE dest [weights]` | O(N+M log M) | Store union |

## Use Cases

### 1. Leaderboards & Rankings
Track user scores with efficient rank queries.

```redis
// Add player scores
ZADD leaderboard 1500 alice
ZADD leaderboard 1200 bob
ZADD leaderboard 1800 charlie
ZADD leaderboard 1300 diana

// Get top 10 players
ZREVRANGE leaderboard 0 9 WITHSCORES  // High to low

// Get bottom 10
ZRANGE leaderboard 0 9 WITHSCORES  // Low to high

// Get player rank (1-indexed)
ZREVRANK leaderboard alice  // Returns: 2 (2nd place)

// Get player score
ZSCORE leaderboard alice  // Returns: 1500

// Update score after game
ZINCRBY leaderboard 150 bob  // Bob scores 150 points

// Get players in score range
ZCOUNT leaderboard 1500 2000  // Count players with score 1500-2000

// Get all with scores
ZRANGEBYSCORE leaderboard 1500 2000 WITHSCORES
```

**Real-world scenario**: Game leaderboards, sports rankings, rating systems.

### 2. Time-based Queues (Priority by Timestamp)
Use timestamp as score for time-ordered processing.

```redis
// Add tasks with timestamp
ZADD task:queue 1705318200 "task:process_payment"
ZADD task:queue 1705318500 "task:send_email"
ZADD task:queue 1705317900 "task:generate_report"

// Get oldest tasks (process in order)
ZRANGE task:queue 0 -1 WITHSCORES

// Get next task to process
ZRANGE task:queue 0 0  // Get first item

// Get tasks within time range
ZRANGEBYSCORE task:queue 1705317000 1705320000

// Process and remove task
ZPOPMIN task:queue  // Get and remove oldest
```

**Real-world scenario**: Job queues, scheduled tasks, event processing.

### 3. Rate Limiting (Sliding Window)
Track requests using timestamp as score.

```redis
// Log requests
ZADD rate:user:123 1705318200 "req:1"
ZADD rate:user:123 1705318205 "req:2"
ZADD rate:user:123 1705318210 "req:3"

// Remove old requests (older than 60 seconds)
current_time = 1705318270
ZREMRANGEBYSCORE rate:user:123 0 (current_time - 60)

// Count requests in window
ZCARD rate:user:123  // Requests in last 60 seconds

// Check if rate limited
if ZCARD rate:user:123 > 10:
  # Rate limit exceeded
```

**Real-world scenario**: API rate limiting, DDoS protection, activity throttling.

### 4. Range Queries (Score-based Filtering)
Find items within numeric ranges.

```redis
// Store products with price as score
ZADD prices:laptop 799.99 "dell:xps"
ZADD prices:laptop 999.99 "apple:macbook"
ZADD prices:laptop 1299.99 "premium:thinkpad"
ZADD prices:laptop 599.99 "budget:asus"

// Find laptops in price range
ZRANGEBYSCORE prices:laptop 600 1000

// Count in range
ZCOUNT prices:laptop 600 1000

// Get highest priced
ZREVRANGE prices:laptop 0 0 WITHSCORES

// Get lowest priced
ZRANGE prices:laptop 0 0 WITHSCORES
```

**Real-world scenario**: Product filtering, range-based search, price comparison.

### 5. Trending/Popularity Scoring
Track popularity metrics efficiently.

```redis
// Articles with view count as score
ZADD trending:articles 1500 "article:redis-guide"
ZADD trending:articles 3200 "article:python-tips"
ZADD trending:articles 900 "article:database-design"

// Get trending articles
ZREVRANGE trending:articles 0 9 WITHSCORES  // Top 10

// Increment views
ZINCRBY trending:articles 50 "article:redis-guide"

// Get articles with high engagement
ZCOUNT trending:articles 2000 10000
```

**Real-world scenario**: Trending content, popularity rankings, engagement metrics.

### 6. Autocomplete (Lexicographic Ordering)
Use same score for lexicographic sorting.

```redis
// Store with same score for alphabetical order
ZADD autocomplete:users 0 "alice"
ZADD autocomplete:users 0 "alice_johnson"
ZADD autocomplete:users 0 "alice_smith"
ZADD autocomplete:users 0 "bob"

// Get items lexicographically
ZRANGEBYLEX autocomplete:users "[a" "[c"

// Prefix match
ZCOUNTBYLEX autocomplete:users "[alice" "[alice_z"
```

### 7. Geographic Proximity (Distance as Score)
Store distances as scores for sorted access.

```redis
// Store nearby restaurants (distance in meters as score)
ZADD restaurants:nearby 250 "burger_king:main"
ZADD restaurants:nearby 450 "mcdonalds:central"
ZADD restaurants:nearby 180 "subway:downtown"
ZADD restaurants:nearby 600 "kfc:airport"

// Find closest restaurants
ZRANGE restaurants:nearby 0 4 WITHSCORES

// Get restaurants within 500m
ZRANGEBYSCORE restaurants:nearby 0 500
```

## Advanced Patterns

### Intersection with Weights
```redis
// Find users common to multiple interests with weighted scores
ZINTERSTORE result 2 interests:tech interests:startup WEIGHTS 2 1
// Users interested in both tech AND startups, weighted 2:1
```

### Union for Aggregation
```redis
// Combine scores from multiple sources
ZUNIONSTORE aggregate 2 user:views user:engagement WEIGHTS 0.3 0.7
// Combined score = views*0.3 + engagement*0.7
```

### Blocking Operations for Workers
```redis
// Wait for next task
BZPOPMIN task:queue 0  // Blocks until task available
```

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Add member | O(log M) | M = number of members |
| Remove member | O(log M) | |
| Get by rank | O(log M + N) | N = elements returned |
| Get by score | O(log M + N) | N = elements in range |
| Get score | O(1) | Direct lookup |
| Increment | O(log M) | Rebalancing required |
| Range query | O(log M + N) | Efficient skip list |
| Intersection | O(N*K log M) | N = smallest set, K = sets |

## Memory Efficiency

- Skip list overhead: ~40 bytes per element
- Efficient for 1M+ members
- Better than hashes for ranked access
- More memory than sets but enables sorting

## Limitations

- **Floating-point scores**: Cannot store integer-only data types
- **Score precision**: Limited to IEEE 754 double precision (~15 digits)
- **No negative infinity bounds**: Use -inf and +inf strings instead
- **No member-to-member relationships**: Cannot query by member similarity

## Best Practices

1. **Use ZRANGE for top-N queries**: O(log M + N) is efficient
2. **Store timestamps as integer milliseconds**: Better precision than float
3. **Use ZREMRANGEBYRANK for cleanup**: Remove oldest entries efficiently
4. **Index scores strategically**: Pre-normalize scores for consistent ranges
5. **Combine with other structures**: Use hashes for rich attributes
6. **Watch for concurrent updates**: Increment operations can race

## Common Pitfalls

1. **Floating-point precision**: 1.1 + 2.2 ≠ 3.3 exactly
2. **Large result sets**: ZRANGE returning millions of items blocks Redis
3. **Expensive intersections**: Multiple large set intersections are slow
4. **Memory with many sets**: Each member duplicated across sets
5. **Unbounded growth**: ZSets without cleanup consume increasing memory
