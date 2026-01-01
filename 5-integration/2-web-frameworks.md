# Web Frameworks - Django, FastAPI, Flask

## Overview

Redis integration with web frameworks enables caching, sessions, and rate limiting at scale.

## Django Integration

### Setup

```python
# settings.py
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            "CONNECTION_POOL_KWARGS": {"max_connections": 50},
        }
    }
}

# Sessions in Redis
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

### Usage

```python
from django.views.decorators.cache import cache_page
from django.core.cache import cache

# Cache view output
@cache_page(60 * 15)  # 15 minutes
def product_list(request):
    products = Product.objects.all()
    return render(request, 'products.html', {'products': products})

# Manual cache management
def get_user_profile(user_id):
    cache_key = f'user_profile:{user_id}'
    profile = cache.get(cache_key)
    
    if profile is None:
        profile = UserProfile.objects.get(user_id=user_id)
        cache.set(cache_key, profile, timeout=3600)
    
    return profile

# Invalidate on update
def update_profile(user_id, data):
    profile = UserProfile.objects.get(user_id=user_id)
    profile.update(**data)
    
    # Invalidate cache
    cache.delete(f'user_profile:{user_id}')
```

---

## FastAPI Integration

### Setup

```python
from fastapi import FastAPI
import redis.asyncio as redis

app = FastAPI()

# Global Redis connection
redis_client = None

@app.on_event("startup")
async def startup():
    global redis_client
    redis_client = await redis.from_url("redis://localhost:6379")

@app.on_event("shutdown")
async def shutdown():
    await redis_client.close()
```

### Caching Decorator

```python
from functools import wraps
import json

def cache_response(ttl: int = 3600):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key from function name and args
            cache_key = f"{func.__name__}:{str(args)}:{str(kwargs)}"
            
            # Try cache first
            cached = await redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # Cache miss: call function
            result = await func(*args, **kwargs)
            
            # Store in cache
            await redis_client.setex(cache_key, ttl, json.dumps(result))
            return result
        return wrapper
    return decorator

# Usage
@app.get("/products/{product_id}")
@cache_response(ttl=3600)
async def get_product(product_id: int):
    return await db.get_product(product_id)
```

---

## Flask Integration

### Setup with Flask-Caching

```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)
cache = Cache(app, config={
    'CACHE_TYPE': 'redis',
    'CACHE_REDIS_URL': 'redis://localhost:6379/0'
})

# Or with custom Redis connection
from redis import Redis
app.config['CACHE_REDIS_CONN'] = Redis(host='localhost', port=6379)
```

### Usage

```python
@app.route('/users/<int:user_id>')
@cache.cached(timeout=3600)
def get_user(user_id):
    user = db.query(User).get(user_id)
    return jsonify(user)

@app.route('/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    # Update database
    user = db.query(User).get(user_id)
    user.update(request.json)
    db.commit()
    
    # Invalidate cache
    cache.delete(f'get_user-{user_id}')
    
    return jsonify(user)
```

---

## Session Management

### Redis Sessions (All Frameworks)

```python
# Store user sessions in Redis instead of database
session_data = {
    'user_id': 123,
    'username': 'alice',
    'last_activity': time.time(),
    'permissions': ['read', 'write']
}

# Set session with auto-expiry
r.setex(f'session:{session_id}', 86400, json.dumps(session_data))

# Retrieve session
session = json.loads(r.get(f'session:{session_id}'))

# Update last activity (touch)
r.expire(f'session:{session_id}', 86400)
```

---

## Rate Limiting

### Middleware Implementation

```python
from time import time
from functools import wraps

def rate_limit(max_requests: int = 100, window: int = 60):
    """Rate limit decorator"""
    def decorator(func):
        @wraps(func)
        def wrapper(request, *args, **kwargs):
            client_ip = request.remote_addr
            key = f'rate_limit:{client_ip}:{func.__name__}'
            
            current = r.incr(key)
            if current == 1:
                r.expire(key, window)
            
            if current > max_requests:
                return {'error': 'Rate limit exceeded'}, 429
            
            return func(request, *args, **kwargs)
        return wrapper
    return decorator

# Usage
@app.route('/api/search')
@rate_limit(max_requests=10, window=60)
def search(request):
    # Can only call 10 times per minute per IP
    pass
```

---

## Best Practices

### DO: Key Naming Convention

```python
# ✅ GOOD: Clear, hierarchical keys
cache_key = f'user:{user_id}:profile'
cache_key = f'product:{product_id}:details'
cache_key = f'search:query:{query_hash}'
```

### DO: Version Cache Keys

```python
# ✅ GOOD: Handle schema changes
CACHE_VERSION = '1'
cache_key = f'v{CACHE_VERSION}:user:{user_id}'

# When schema changes, bump version
CACHE_VERSION = '2'
# Old cached data automatically invalidated
```

### DON'T: Cache Sensitive Data

```python
# ❌ BAD: Caching passwords or tokens
r.set(f'user:{user_id}:password', hashed_password)

# ✅ GOOD: Cache only public data
r.set(f'user:{user_id}:profile', public_info)
```

---

## Common Mistakes

### Mistake 1: No TTL on Cache Keys
**Problem**: Memory bloat, stale data indefinitely
**Solution**: Always set reasonable TTLs with setex()

### Mistake 2: Cache Stampede
**Problem**: Multiple requests fetch DB when cache expires
**Solution**: Use locks or pre-fetch with background tasks

### Mistake 3: Serialization Overhead
**Problem**: JSON serialization too slow
**Solution**: Use msgpack for faster serialization

### Mistake 4: Not Handling Cache Misses
**Problem**: Crashes if Redis unavailable
**Solution**: Fallback to database, try-catch blocks

---

## Next Steps

1. **Learn [Message Queues](3-message-queues.md)** for background jobs
2. **Explore [Real-time Systems](4-realtime.md)** for live updates
3. **Review [Caching Strategies](6-caching.md)** for optimization

## Resources

- [Django Redis Docs](https://niwinz.github.io/django-redis/)
- [FastAPI Caching](https://fastapi.tiangolo.com/advanced/caching/)
- [Flask Caching](https://flask-caching.readthedocs.io/)
