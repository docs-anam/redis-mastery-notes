# Caching Strategies - Multi-Layer Optimization

## Overview

Multi-layer caching dramatically improves application performance by reducing latency and database load.

## Caching Layers

```
┌─────────────────────────────────────┐
│     Browser Cache (HTTP Cache)       │ 5min-1yr
├─────────────────────────────────────┤
│    CDN Cache (CloudFront, Akamai)    │ 1min-1day
├─────────────────────────────────────┤
│   Application Cache (Redis)          │ 1min-1hour
├─────────────────────────────────────┤
│   Query Cache (Redis)                │ 5min-30min
├─────────────────────────────────────┤
│   Database (PostgreSQL, MySQL)       │ Persistent
└─────────────────────────────────────┘
```

---

## Layer 1: Browser Cache

### HTTP Cache Headers

```python
from flask import Flask, make_response

app = Flask(__name__)

@app.route('/api/products/<product_id>')
def get_product(product_id):
    product = db.query(Product).get(product_id)
    
    response = make_response(jsonify(product))
    # Cache for 1 hour in browser
    response.cache_control.max_age = 3600
    response.cache_control.public = True
    
    return response
```

**Benefits**: 0ms latency, zero server load
**Trade-off**: Requires cache invalidation strategy

---

## Layer 2: Application Cache

### Cache-Aside Pattern

```python
def get_user_with_cache(user_id):
    # Try cache first
    cache_key = f'user:{user_id}'
    user = r.get(cache_key)
    
    if user:
        return json.loads(user)
    
    # Cache miss: fetch from DB
    user = db.query(User).get(user_id)
    
    # Store in cache (1 hour)
    r.setex(cache_key, 3600, json.dumps(user))
    
    return user
```

**Benefits**: Simple, works with any backend
**Trade-off**: Stale data possible

### Write-Through Cache

```python
def update_user(user_id, data):
    # Update database first
    user = db.query(User).get(user_id)
    user.update(**data)
    db.commit()
    
    # Update cache immediately
    cache_key = f'user:{user_id}'
    r.setex(cache_key, 3600, json.dumps(user))
    
    # Invalidate related caches
    r.delete(f'user:{user_id}:preferences')
    r.delete('users:list')
    
    return user
```

**Benefits**: Consistency, always fresh
**Trade-off**: Slower writes

---

## Layer 3: Query Cache

### Result Set Caching

```python
def get_top_products(category, limit=10):
    cache_key = f'products:top:{category}:{limit}'
    
    # Try cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Execute query
    products = db.query(Product)\
        .filter(Product.category == category)\
        .order_by(Product.sales.desc())\
        .limit(limit)\
        .all()
    
    # Cache query result
    r.setex(cache_key, 1800, json.dumps([p.to_dict() for p in products]))
    
    return products
```

**Benefits**: Large performance gains for expensive queries
**Trade-off**: Tight coupling to queries

---

## Cache Warming

### Pre-populate Cache

```python
def warm_cache():
    """Pre-populate hot data at startup"""
    
    # Cache popular products
    products = db.query(Product).filter(Product.popular == True).all()
    for product in products:
        key = f'product:{product.id}'
        r.setex(key, 86400, json.dumps(product.to_dict()))
    
    # Cache feature flags
    flags = db.query(FeatureFlag).all()
    for flag in flags:
        key = f'feature:{flag.name}'
        r.set(key, flag.enabled)
    
    print(f"Warmed {len(products)} products")

# Run on startup
@app.before_first_request
def startup():
    warm_cache()
```

---

## Cache Invalidation

### Strategies

```python
class CacheManager:
    def __init__(self, r):
        self.r = r
    
    def invalidate_user(self, user_id):
        """Invalidate all user-related caches"""
        patterns = [
            f'user:{user_id}:*',
            f'users:list',
            f'user:{user_id}:recommendations:*'
        ]
        
        for pattern in patterns:
            keys = self.r.keys(pattern)
            if keys:
                self.r.delete(*keys)
    
    def invalidate_tag(self, tag):
        """Invalidate by tag"""
        keys = self.r.smembers(f'cache:tag:{tag}')
        if keys:
            self.r.delete(*keys)
        self.r.delete(f'cache:tag:{tag}')
    
    def set_with_tag(self, key, value, ttl, tag):
        """Set cache with tag"""
        self.r.setex(key, ttl, value)
        self.r.sadd(f'cache:tag:{tag}', key)
        self.r.expire(f'cache:tag:{tag}', ttl)

# Usage
cache = CacheManager(r)
cache.set_with_tag('product:123', product_data, 3600, 'products')
cache.invalidate_tag('products')  # Invalidate all product caches
```

---

## Conditional Requests

### ETags for Cache Validation

```python
from hashlib import md5

@app.route('/api/user/<user_id>')
def get_user(user_id):
    user = get_user_with_cache(user_id)
    
    # Generate ETag
    etag = md5(json.dumps(user).encode()).hexdigest()
    
    # Client sends If-None-Match header with previous ETag
    if request.headers.get('If-None-Match') == etag:
        return '', 304  # Not Modified
    
    response = jsonify(user)
    response.headers['ETag'] = etag
    return response
```

**Benefits**: Saves bandwidth on unchanged data
**Trade-off**: Additional computation

---

## Best Practices

### DO: Set Reasonable TTLs

```python
# ✅ GOOD: Different TTLs for different data
r.setex('user:profile:123', 3600, data)      # 1 hour
r.setex('product:featured', 300, data)       # 5 minutes
r.setex('config:flags', 600, data)           # 10 minutes
```

### DO: Cache Negative Results

```python
# ✅ GOOD: Cache "not found" too
user = db.query(User).get(user_id)
if user:
    r.setex(f'user:{user_id}', 3600, json.dumps(user))
else:
    r.setex(f'user:{user_id}:notfound', 600, 'null')
```

### DON'T: Cache Without TTL

```python
# ❌ BAD: Data never expires
r.set(f'user:{user_id}', json.dumps(user))

# ✅ GOOD: Always set expiration
r.setex(f'user:{user_id}', 3600, json.dumps(user))
```

---

## Performance Impact

| Scenario | No Cache | With Cache | Improvement |
|----------|----------|-----------|------------|
| Cache hit | - | 1ms | - |
| Single miss | 100ms | 100ms | 0% |
| 80% hit rate | 100ms | 20ms | 5x |
| 95% hit rate | 100ms | 5ms | 20x |

---

## Common Mistakes

### Mistake 1: Cache Stampede
**Problem**: Many requests query DB when cache expires
**Solution**: Use locks or probabilistic expiration

### Mistake 2: Cache Bloat
**Problem**: Memory fills with unused keys
**Solution**: Implement eviction policy, set maxmemory

### Mistake 3: Uncached Expensive Queries
**Problem**: Database overloaded
**Solution**: Identify and cache slow queries

### Mistake 4: No Cache Metrics
**Problem**: Can't see cache effectiveness
**Solution**: Track hit ratio, evictions, memory

---

## Next Steps

1. **Review [Web Frameworks](2-web-frameworks.md)** for integration
2. **Study [Performance](../4-performance/1-intro.md)** for optimization
3. **Explore [Monitoring](../4-performance/5-monitoring.md)** for production

## Resources

- [Cache Invalidation Patterns](https://www.codementor.io/@mitchartemis/cache-invalidation-what-when-and-where-3a63phfxflu)
- [Redis Caching Best Practices](https://redis.io/topics/client-side-caching)
- [HTTP Caching Guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
