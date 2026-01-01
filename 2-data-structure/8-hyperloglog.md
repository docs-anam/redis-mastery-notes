# HyperLogLog in Redis

## Overview
HyperLogLog is a probabilistic data structure for estimating the cardinality (number of unique elements) of a set with minimal memory usage (~12KB regardless of set size). It trades accuracy for space efficiency with ~0.81% standard error, ideal for counting unique visitors, searches, and large-scale deduplication.

## Key Characteristics
- **Memory Efficient**: Constant ~12KB memory regardless of cardinality
- **Probabilistic**: Approximate counts, not exact values
- **Fast**: O(1) time complexity for additions and merges
- **Error Rate**: Standard error of ~0.81%
- **Mergeable**: Can combine multiple HyperLogLogs
- **Unique Elements**: Designed specifically for cardinality estimation

## Common Commands

### Basic Operations
| Command | Complexity | Description |
|---------|-----------|-------------|
| `PFADD key element [element ...]` | O(1) | Add elements to HyperLogLog |
| `PFCOUNT key [key ...]` | O(N) | Get cardinality estimate |
| `PFMERGE dest source [source ...]` | O(N) | Merge multiple HyperLogLogs |

### Key Points
- PFADD returns 1 if HLL was modified, 0 if not
- PFCOUNT can estimate for multiple keys at once
- PFMERGE combines estimates from multiple HLLs

## Use Cases

### 1. Unique Daily Visitors
Track unique users visiting website per day with minimal memory.

```redis
// Add daily visitors
PFADD visitors:2024-01-15 "192.168.1.1"
PFADD visitors:2024-01-15 "192.168.1.2"
PFADD visitors:2024-01-15 "192.168.1.3"
PFADD visitors:2024-01-15 "192.168.1.1"  // Duplicate, still O(1)

// Get estimated unique visitors today
PFCOUNT visitors:2024-01-15  // Returns: ~3 (actual: 3)

// Multiple days
PFCOUNT visitors:2024-01-15 visitors:2024-01-16 visitors:2024-01-17
// Returns: ~450 (union of all three days)

// Get month total
PFMERGE visitors:month:2024-01 visitors:2024-01-01 visitors:2024-01-02 ... visitors:2024-01-31
PFCOUNT visitors:month:2024-01  // Month's unique visitors
```

**Real-world scenario**: Web analytics, unique visitor tracking, traffic analysis.

### 2. Unique Search Queries
Track how many unique search terms users searched.

```redis
// Log searches
PFADD searches:2024-01-15 "redis tutorial"
PFADD searches:2024-01-15 "python guide"
PFADD searches:2024-01-15 "redis tutorial"  // Duplicate, still efficient
PFADD searches:2024-01-15 "javascript"
PFADD searches:2024-01-15 "python guide"

// Estimated unique queries
PFCOUNT searches:2024-01-15  // Returns: ~3 (actual: 3)

// Compare search volume across days
PFCOUNT searches:2024-01-14 searches:2024-01-15 searches:2024-01-16

// Weekly search diversity
PFMERGE searches:week:1 searches:monday searches:tuesday searches:wednesday
PFCOUNT searches:week:1
```

**Real-world scenario**: Search analytics, trending topics, unique queries.

### 3. Unique IP Addresses & Bot Detection
Count unique IPs accessing your service.

```redis
// Track IPs per endpoint
PFADD ips:api:users:get "192.168.1.1"
PFADD ips:api:users:get "192.168.1.2"
PFADD ips:api:users:get "203.0.113.5"    // Potential bot
PFADD ips:api:users:get "198.51.100.10"  // Potential bot

// Estimated unique IPs
PFCOUNT ips:api:users:get  // Returns: ~4

// Track suspicious endpoints
PFADD ips:api:auth:login "203.0.113.5"
PFADD ips:api:auth:login "203.0.113.5"
PFADD ips:api:auth:login "203.0.113.5"
// If many requests from few IPs, likely attack

// Compare IP diversity across endpoints
PFCOUNT ips:api:users:get ips:api:products:list ips:api:auth:login
```

**Real-world scenario**: Bot detection, DDoS analysis, API monitoring.

### 4. Unique Hashtags & Topics
Track diversity of topics in social media.

```redis
// Instagram hashtags
PFADD hashtags:2024-01 "#redis" "#database" "#cache"
PFADD hashtags:2024-01 "#python" "#redis" "#optimization"
PFADD hashtags:2024-01 "#coding" "#beginner" "#tutorial"

// Estimated unique hashtags
PFCOUNT hashtags:2024-01  // Returns: ~7 (actual: 8)

// Topic diversity per day
PFCOUNT hashtags:2024-01-15 hashtags:2024-01-16 hashtags:2024-01-17

// Trending topics
PFMERGE trending:week hashtags:monday hashtags:tuesday hashtags:wednesday
PFCOUNT trending:week
```

**Real-world scenario**: Social media analytics, hashtag tracking, trending topics.

### 5. Unique Products Viewed
Track how many different products users viewed.

```redis
// User views products
PFADD user:123:viewed:products "product:101"
PFADD user:123:viewed:products "product:102"
PFADD user:123:viewed:products "product:103"
PFADD user:123:viewed:products "product:101"  // Reviewed, still O(1)

// How many unique products viewed?
PFCOUNT user:123:viewed:products  // Returns: ~3

// Compare browsing across users
PFCOUNT user:123:viewed:products user:124:viewed:products user:125:viewed:products

// Store-wide metrics
PFMERGE store:unique:products user:123:viewed:products user:124:viewed:products ...
PFCOUNT store:unique:products  // All unique products viewed by any user
```

**Real-world scenario**: E-commerce browsing, product discovery analytics, user profiling.

### 6. Unique Active Users
Count distinct users during time period.

```redis
// Log active users per hour
PFADD active:users:2024-01-15:08 "user:101" "user:102" "user:103"
PFADD active:users:2024-01-15:09 "user:102" "user:104" "user:105"
PFADD active:users:2024-01-15:10 "user:101" "user:105" "user:106"

// Hourly active users
PFCOUNT active:users:2024-01-15:08  // Returns: ~3

// Daily active users (union)
PFCOUNT active:users:2024-01-15:08 active:users:2024-01-15:09 active:users:2024-01-15:10
// Returns: ~5

// Monthly active users
PFMERGE active:users:month:2024-01 active:users:2024-01-15 active:users:2024-01-16 ...
PFCOUNT active:users:month:2024-01
```

**Real-world scenario**: User engagement metrics, DAU/MAU calculations, concurrent users.

## Accuracy & Error Rate

```
HyperLogLog accuracy:
- Standard error: ±0.81%
- For 1 million items: ±8,100 items error
- For 1 billion items: ±8.1 million items error

Comparison with alternatives:
- Exact set (SADD): O(N) memory, O(1) operations
- HyperLogLog (PFADD): O(1) memory, O(1) operations, ±0.81% error
- Trade: Accuracy for massive memory savings
```

## Memory Efficiency

```
Memory comparison for different cardinalities:
- Exact set 1,000 items: ~50KB
- HyperLogLog 1,000 items: ~12KB

- Exact set 1 million items: ~50MB
- HyperLogLog 1 million items: ~12KB

- Exact set 1 billion items: ~50GB
- HyperLogLog 1 billion items: ~12KB
```

## Limitations

- **Approximate only**: Cannot retrieve individual elements
- **Non-reversible**: Cannot get exact members from HLL
- **No cardinality adjustment**: Cannot reduce estimated count
- **Merge creates new key**: PFMERGE creates new HyperLogLog
- **No subtracting members**: Cannot remove individual elements

## Advanced Patterns

### Time-series Aggregation
```redis
// Hourly unique IPs
PFADD ips:hour:1 ... (100 users)
PFADD ips:hour:2 ... (120 users)
PFADD ips:hour:3 ... (110 users)

// Daily unique (union)
PFMERGE ips:daily ips:hour:1 ips:hour:2 ... ips:hour:24
PFCOUNT ips:daily  // Unique IPs that day
```

### Multi-dimensional Tracking
```redis
// Track by region and country
PFADD visits:us:2024-01-15 <user_ids>
PFADD visits:uk:2024-01-15 <user_ids>
PFADD visits:ca:2024-01-15 <user_ids>

// Regional comparison
PFCOUNT visits:us:2024-01-15 visits:uk:2024-01-15
```

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Add element | O(1) | Constant time |
| Get cardinality | O(1) | Fast estimation |
| Merge | O(N) | N = number of HLLs merged |
| Memory | O(1) | Always ~12KB |

## Best Practices

1. **Use for cardinality only**: Don't use PFADD if you need individual elements
2. **Accept approximation**: Plan for ±0.81% error in decisions
3. **Use separate HLLs by dimension**: Time, region, type, etc.
4. **Monitor memory**: HLLs are small but can accumulate
5. **Merge strategically**: Create daily/weekly summaries
6. **Pipeline operations**: Batch PFADD calls for efficiency
7. **Combine with other structures**: Use Sets for exact, HLL for approximate

## Common Pitfalls

1. **Expecting exact count**: HyperLogLog gives estimates, not exact values
2. **Subtracting cardinalities**: Cannot reliably subtract HLL estimates
3. **Over-precision**: Don't make critical decisions on ±1% estimates
4. **No element retrieval**: Cannot get individual members back
5. **Assuming merge is cheap**: PFMERGE creates new key, uses memory
6. **Too many HLLs**: Remember each HLL takes ~12KB even if empty
