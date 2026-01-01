# When Do You Need Redis?

## Decision Framework

Redis is powerful, but it's not a universal solution. This guide helps you determine if Redis is the right choice for your use case.

## Use Redis When...

### 1. You Need Speed (Cache Layer)
**Symptom**: Your application is slow because the database can't keep up
**Solution**: Add Redis as a cache layer

```
Without Redis:
User Request → Database Query (1-100ms) → Response (101-100ms)

With Redis:
User Request → Redis Cache (1μs) HIT → Response (1μs)
              → Database Query (100ms) MISS → Response (100ms)
```

**Benefit**: 90% of requests hit cache, 100x faster average response time

**Examples:**
- Frequently accessed user profiles
- Product catalogs
- API responses
- Query results

### 2. You Need to Store Sessions
**Symptom**: Scaling web servers requires shared session storage

```
Architecture:
Multiple Web Servers → Redis Session Store ← All servers access same sessions
```

**Benefits:**
- Sessions survive server restarts
- Shared across load-balanced servers
- Automatic expiration (TTL)
- Very fast access

**Example:**
```python
# Store user session
r.setex(f"session:{user_id}", 3600, user_data)

# All servers can access the same session
session = r.get(f"session:{user_id}")
```

### 3. You Need Real-Time Metrics
**Symptom**: Need to track stats without hitting the database every time

```
Redis Analytics:
INCR commands → Instant counters
SET commands → Snapshots
ZADD commands → Rankings
SADD commands → Unique tracking
```

**Benefits:**
- No database round-trips
- Instant results
- Aggregatable
- Atomic operations

**Examples:**
- Page view counters
- Active user counts
- Leaderboards
- Trending topics
- Real-time dashboards

### 4. You Need Rate Limiting
**Symptom**: Must limit requests per user/IP

```python
# Check rate limit
requests = r.incr(f"rate:{user_id}")
if requests > 100:
    return "Too many requests"

# Auto-expire after window
r.expire(f"rate:{user_id}", 3600)
```

### 5. You Need Pub/Sub Messaging
**Symptom**: Need real-time event distribution

```
Event Publisher → Redis Pub/Sub → Multiple Subscribers
                     (instant)
```

**Use Cases:**
- Live notifications
- Real-time chat
- Cache invalidation
- Event broadcasting

### 6. You Need Distributed Coordination
**Symptom**: Multiple services need to coordinate

```
Distributed Lock Example:
Service A: LOCK → Do work → UNLOCK
Service B: Wait for LOCK release → Do work

Rate Limiter:
User: REQUEST → Check rate:user → ACCEPT/REJECT
```

## Don't Use Redis When...

### 1. Your Data is Too Large
**Problem**: Redis data must fit in RAM

```
Guideline:
Small dataset (< 1GB):     Redis is ideal
Medium dataset (1-10GB):   Possible with careful eviction
Large dataset (> 10GB):    Use PostgreSQL/MongoDB instead
Huge dataset (> 100GB):    Definitely not Redis
```

**Solutions for large data:**
- PostgreSQL, MongoDB, Cassandra
- Use data warehouses (Snowflake)
- Elasticsearch for search

### 2. You Need Complex Transactions
**Problem**: Redis doesn't support multi-step ACID transactions

```
Bad for Redis:
1. Check account balance
2. Deduct money
3. Update inventory
4. Record transaction

Better: PostgreSQL with ACID guarantees
```

### 3. You Need Complex Queries
**Problem**: Redis doesn't support JOINs or complex filtering

```
Bad for Redis:
SELECT * FROM users 
WHERE age > 18 AND city = 'NYC' 
AND created_at > '2024-01-01'

Better: PostgreSQL with advanced queries
```

### 4. You Can't Lose Data
**Problem**: Redis data is volatile even with persistence

```
Data Loss Scenarios:
- Server crashes (RDB) → lose updates since last snapshot
- Network split → data loss possible
- Application crash → unsaved data lost

If you can't lose data: Use database with WAL (PostgreSQL)
```

### 5. You Have Enormous Concurrency
**Problem**: Single Redis instance has limits

```
Single Instance Capacity:
~100,000 operations/second on typical hardware

If you need more: Use Redis Cluster or message queue (Kafka)
```

## Use Case Decision Tree

```
START: Do you need fast access to data?
│
├─ NO → Use traditional database (PostgreSQL/MongoDB)
│
└─ YES → Can the entire dataset fit in RAM?
    │
    ├─ NO → Use database or distributed cache (Memcached)
    │
    └─ YES → Do you need complex transactions?
        │
        ├─ YES → Use database + Redis for caching
        │
        └─ NO → Do you need complex queries?
            │
            ├─ YES → Use database + Redis for hot data
            │
            └─ NO → Can you afford to lose data?
                │
                ├─ YES → Use Redis as cache (no persistence)
                │
                └─ NO → Use Redis with persistence (RDB/AOF)
```

## Common Use Cases with Solutions

### Use Case 1: E-Commerce Site
**Problem**: User browsing is slow, database is overloaded

**Solution:**
```
Product Page Request:
1. Check Redis for product data (99% hit)
2. On miss, fetch from database
3. Cache for 1 hour
4. Invalidate on updates

Result: Database load drops 90%+
```

### Use Case 2: Real-Time Chat
**Problem**: Need to broadcast messages instantly

**Solution:**
```
Redis Pub/Sub:
- User sends message
- Publish to room channel
- All connected users get message instantly
- No database hit needed
```

### Use Case 3: Leaderboard
**Problem**: Thousands of score updates per second

**Solution:**
```
Redis Sorted Set:
ZADD leaderboard {user1: 1000, user2: 950}
ZREVRANGE leaderboard 0 9  # Top 10
ZREVRANK leaderboard user1 # User's rank

Instant results, scales to millions of users
```

### Use Case 4: Rate Limiting API
**Problem**: Must limit requests per user

**Solution:**
```python
# Fast, accurate rate limiting with Redis
key = f"rate:{user_id}:{minute}"
r.incr(key)
r.expire(key, 60)
if r.get(key) > 100:
    return 429 Limit Exceeded
```

### Use Case 5: Session Management
**Problem**: Sharing sessions across 10 web servers

**Solution:**
```python
# Store in Redis
r.setex(f"session:{sid}", 3600, session_json)

# Any server can read from Redis
session = r.get(f"session:{sid}")

# Scales to unlimited web servers
```

### Use Case 6: Real-Time Analytics
**Problem**: Dashboard needs instant stats

**Solution:**
```
Redis Counters:
- Increment counters in real-time
- Group by category, hour, day
- Instant aggregation
- Zero database load
```

## Hybrid Approach (Best Practice)

Most production systems use **Redis + Database**:

```
┌─────────────┐
│ Application │
└──────┬──────┘
       │
       ├─→ Cache Miss? → Database
       │   (Fresh data)
       │
       ├─→ Cache Hit → Redis
       │   (1μs response)
       │
       └─→ Write? → Both (DB + Redis)
           (Consistency)
```

**Pattern:**
- **Read**: Check Redis first, fall back to database
- **Write**: Write to database, update Redis
- **Expiration**: Set TTL on cached items

## Summary Table

| Use Case | Use Redis? | Alternative |
|----------|-----------|------------|
| Caching | ✅ Yes | Memcached |
| Sessions | ✅ Yes | Database |
| Counters | ✅ Yes | Database (slow) |
| Leaderboards | ✅ Yes | Database (slow) |
| Pub/Sub | ✅ Yes | RabbitMQ, Kafka |
| Rate Limiting | ✅ Yes | Database (slow) |
| Complex Transactions | ❌ No | PostgreSQL |
| Complex Queries | ❌ No | PostgreSQL |
| Large Datasets (>100GB) | ❌ No | PostgreSQL, MongoDB |
| Write-heavy + Durability | ❌ No | PostgreSQL |

## Performance Expectations

### Where Redis Shines
```
Operation        Latency    Throughput
String SET/GET   1 μs       100k+ ops/sec
List LPUSH       1 μs       100k+ ops/sec
Set operations   1 μs       100k+ ops/sec
Counter INCR     1 μs       100k+ ops/sec
Sorted Set ops   1 μs       50k+ ops/sec
```

### Where It Struggles
```
Operation           Issue
Complex Query       Not supported
JOIN operation      Not supported
Multi-step TX       Limited
100GB dataset       Won't fit in RAM
Millisecond TTL     Not reliable
```

## Recommended Reading

- [Installation Guide](3-install-redis.md)
- [Configuration & Tuning](4-configuration.md)
- [Strings Data Type](6-strings.md)
- [Transactions & Pipelines](8-transaction.md)
- [Persistence](15-persistence.md)
- [Security](14-security.md)

## Key Takeaway

**Redis is perfect for speed-critical, in-memory data with a size that fits in RAM.**

If you need:
- ✅ Lightning-fast caching
- ✅ Session storage
- ✅ Real-time metrics
- ✅ Message queues
- ✅ Distributed coordination

**Then Redis is your answer.**

If you need:
- ❌ Complex queries
- ❌ Large datasets (>100GB)
- ❌ ACID transactions
- ❌ Guaranteed durability

**Then use a traditional database (PostgreSQL, MongoDB, etc.)**

The best approach: **Use both! Redis for speed, PostgreSQL for reliability.**
