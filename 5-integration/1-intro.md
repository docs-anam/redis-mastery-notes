# Integration - Building Real Applications

## Overview

Integration patterns show how to combine Redis with application frameworks, databases, and services in production systems.

## Section Topics

1. **[Web Frameworks](2-web-frameworks.md)** - Django, FastAPI, Flask integration
2. **[Message Queues](3-message-queues.md)** - Task scheduling, event processing
3. **[Real-time Systems](4-realtime.md)** - WebSockets, live updates, notifications
4. **[Search & Analytics](5-search.md)** - Elasticsearch, data aggregation
5. **[Caching Strategies](6-caching.md)** - Multi-layer caching, cache invalidation

---

## Integration Patterns

### Pattern 1: Cache-Aside (Lazy Loading)

```python
def get_user(user_id):
    # Check cache first
    cached = r.get(f'user:{user_id}')
    if cached:
        return json.loads(cached)
    
    # Cache miss: fetch from DB
    user = db.query(f"SELECT * FROM users WHERE id={user_id}")
    
    # Store in cache
    r.setex(f'user:{user_id}', 3600, json.dumps(user))
    return user
```

**Benefits**: Simplicity, high hit rates
**Trade-off**: Stale data until expiration

### Pattern 2: Write-Through

```python
def update_user(user_id, data):
    # Write to DB first
    db.execute(f"UPDATE users SET ... WHERE id={user_id}")
    
    # Then update cache
    r.setex(f'user:{user_id}', 3600, json.dumps(data))
    
    # Invalidate related caches
    r.delete(f'user:{user_id}:profile')
    r.delete('users:list')
```

**Benefits**: Data consistency, immediate updates
**Trade-off**: Slower writes, cache maintenance

### Pattern 3: Cache Invalidation

```python
def update_product(product_id, data):
    # Update database
    db.update_product(product_id, data)
    
    # Invalidate product cache
    r.delete(f'product:{product_id}')
    r.delete(f'product:{product_id}:variants')
    r.delete('products:featured')
    
    # Broadcast to other services
    r.publish('product-updated', product_id)
```

**Benefits**: Consistency, distributed invalidation
**Trade-off**: Complexity, message overhead

---

## Learning Paths

### Path 1: Web Application (3-5 days)

**Focus**: Cache integration with web framework

1. Cache-aside pattern (Django, FastAPI)
2. Session storage
3. Rate limiting middleware
4. Database query optimization

**Outcome**: Fast, scalable web app with Redis caching

### Path 2: Real-time Features (5-7 days)

**Focus**: Live updates and notifications

1. Pub/Sub messaging
2. WebSocket connections
3. Presence tracking
4. Live feeds/leaderboards

**Outcome**: Real-time system with instant updates

### Path 3: Job Processing (4-6 days)

**Focus**: Async task handling

1. Job queue implementation
2. Worker processing
3. Error handling
4. Scheduled tasks

**Outcome**: Scalable background job system

---

## Best Practices

### DO: Use Connection Pooling

```python
# ✅ GOOD: Single pool for entire application
redis_pool = redis.ConnectionPool(...)
redis_client = redis.Redis(connection_pool=redis_pool)

# Share across routes/handlers
@app.route('/api/user/<user_id>')
def get_user(user_id):
    return redis_client.get(f'user:{user_id}')
```

### DO: Handle Connection Failures

```python
# ✅ GOOD: Graceful fallback
def get_cached_data(key):
    try:
        return r.get(key)
    except redis.ConnectionError:
        # Fallback to database
        return get_from_database(key)
```

### DON'T: Cache Everything

```python
# ❌ BAD: Blind caching
r.set('page:view:count', count)  # Always cache?
r.set('session:tempdata:123', data)  # Temporary data?

# ✅ GOOD: Cache strategically
r.setex('user:123:profile', 3600, data)  # Expensive query
r.setex('config:features', 300, flags)    # Stable data
```

---

## Common Integration Mistakes

### Mistake 1: No Cache Invalidation
**Problem**: Stale data serves indefinitely
**Solution**: Set reasonable TTLs, implement invalidation events

### Mistake 2: Single Redis Instance
**Problem**: Single point of failure
**Solution**: Use replication + sentinel for HA

### Mistake 3: Storing Large Objects
**Problem**: Memory bloat, slow serialization
**Solution**: Store references, denormalize carefully

### Mistake 4: No Error Handling
**Problem**: Redis downtime crashes application
**Solution**: Try-catch blocks, fallback to database

### Mistake 5: Unoptimized Queries
**Problem**: Complex cache keys, lookup inefficiency
**Solution**: Key naming conventions, pipelining

---

## Monitoring Integration Health

### Key Metrics

```python
# Cache hit ratio
hits = r.info('stats')['keyspace_hits']
misses = r.info('stats')['keyspace_misses']
hit_ratio = hits / (hits + misses)

# Operation latency
# P99 should be < 5ms for cache hits

# Memory usage
# Track growth trends
memory = r.info('memory')['used_memory_human']
```

### Health Checks

```python
@app.route('/health')
def health_check():
    try:
        r.ping()
        # Check Redis connectivity
        return {'redis': 'healthy'}, 200
    except:
        return {'redis': 'unhealthy'}, 500
```

---

## Next Steps

1. **Choose pattern**: Pick the integration path that fits your use case
2. **Start with [Web Frameworks](2-web-frameworks.md)** for web applications
3. **Explore [Real-time Systems](4-realtime.md)** for live features
4. **Reference [Performance](../4-performance/1-intro.md)** for optimization

---

## Prerequisites

- Sections 1-4 completed (basics through performance)
- Understanding of your application framework
- Production deployment experience
- Monitoring infrastructure in place

---

## Resources

- [Redis Best Practices](https://redis.io/docs/management/admin-guide/)
- [Connection Pooling Guide](https://github.com/redis/redis-py#connection-pools)
- [Cache Invalidation Patterns](https://redis.io/docs/manual/client-side-caching/)
