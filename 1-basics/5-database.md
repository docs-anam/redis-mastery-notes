# Redis Databases

## Overview

Redis supports multiple independent databases within a single instance. By default, Redis has 16 databases (0-15). Each database is a completely separate namespace for keys.

## Database Basics

### Selecting a Database

```redis
# Select database 0 (default)
SELECT 0

# Select database 3
SELECT 3

# Try to select non-existent database (error)
SELECT 20
# Error: ERR DB index is out of range

# Current database info
INFO keyspace
```

### Database Properties

- **Isolated namespaces**: Keys in DB 0 don't exist in DB 1
- **Independent expiration**: TTL timers are per-database
- **Independent operations**: FLUSHDB only affects current DB
- **Shared memory**: All databases share configured maxmemory

## Database Operations

### Switching Databases

```python
import redis

# Connect to database 0 (default)
r = redis.Redis(db=0)

# Connect to database 1
r = redis.Redis(db=1)

# Switch database after connection
r.select(2)
```

### Listing Keys by Database

```redis
# Keys in current database
KEYS *

# Keys matching pattern
KEYS user:*

# Count all keys
DBSIZE

# Database statistics
INFO keyspace
# Output shows keys per database:
# db0:keys=5,expires=2
# db1:keys=3,expires=1
```

### Moving Keys Between Databases

```redis
# Move key from DB 0 to DB 1
MOVE mykey 1

# Move multiple keys (manual approach)
# 1. Get from source DB
SELECT 0
GET mykey
# 2. Set in target DB
SELECT 1
SET mykey value
# 3. Delete from source
SELECT 0
DEL mykey
```

### Clearing Databases

```redis
# Delete all keys in current database
FLUSHDB

# Delete all keys in all databases
FLUSHALL

# Async flush (non-blocking)
FLUSHDB ASYNC
```

## Use Cases for Multiple Databases

### 1. Separate Environments

```redis
# Database 0: Production
SELECT 0
SET user:123 "{name: 'Alice'}"

# Database 1: Development
SELECT 1
SET user:123 "{name: 'Dev User'}"

# Database 2: Testing
SELECT 2
SET user:123 "{name: 'Test User'}"
```

### 2. Logical Data Separation

```redis
# Database 0: User sessions
SELECT 0
SET session:abc123 "{user_id: 456}"

# Database 1: Cache data
SELECT 1
SET cache:products "{...}"

# Database 2: Rate limiting counters
SELECT 2
SET ratelimit:user123 "45"
```

### 3. Tenant Isolation

```redis
# Database 0: Tenant A
SELECT 0
SET customer:1:orders "count:5"

# Database 1: Tenant B
SELECT 1
SET customer:1:orders "count:12"
# Same key, different databases = different values
```

### 4. Multi-Version Data

```redis
# Database 0: Current version
SELECT 0
SET config:v1 "{feature_x: true}"

# Database 1: Previous version
SELECT 1
SET config:v1 "{feature_x: false}"

# Easy rollback: SWAPDB or SELECT different DB
```

## Configuring Databases

### In redis.conf

```conf
# Number of databases (default: 16)
databases 16

# Change to fewer databases
databases 8

# Or more (uses more memory)
databases 32
```

### Changing at Runtime

```bash
# Cannot be changed at runtime
# Must set in config and restart
redis-server --databases 32
```

## SWAPDB Command (Redis 4.0+)

Atomically exchange two databases:

```redis
# Swap database 0 and 1
SWAPDB 0 1

# Useful for:
# 1. Atomic updates
SELECT 0
FLUSHDB
# Restore from backup/rebuild
SET key1 value1
SET key2 value2
# Swap to make live
SELECT 1  # or SWAPDB 1 0

# 2. A/B testing
# Keep DB 0 as stable
# Use DB 1 for testing new data
# SWAPDB when ready
```

## Multi-Database Patterns

### Pattern 1: Cache Isolation

```python
# Production cache: DB 0
cache_prod = redis.Redis(db=0, host='redis.prod')

# Test cache: DB 1
cache_test = redis.Redis(db=1, host='redis.prod')

# Completely isolated caches
cache_prod.set('page:home', html_content)
cache_test.set('page:home', test_html)
```

### Pattern 2: Tenant Databases

```python
def get_tenant_redis(tenant_id):
    # Map tenant to database number
    db = (tenant_id % 16)  # Spread across 16 databases
    return redis.Redis(db=db)

# Tenant 1 → DB 1
r1 = get_tenant_redis(1)

# Tenant 17 → DB 1 (same DB)
r17 = get_tenant_redis(17)

# Each tenant isolated but shares instance
```

### Pattern 3: Environment Separation

```python
import os

def get_redis_db():
    env = os.getenv('ENVIRONMENT', 'dev')
    db_map = {
        'dev': 0,
        'test': 1,
        'staging': 2,
        'prod': 3
    }
    return redis.Redis(db=db_map[env])

# Code works same everywhere
# But uses different database per environment
redis_client = get_redis_db()
```

### Pattern 4: Zero-Downtime Updates

```python
# Current DB (DB 0)
current = redis.Redis(db=0)

# Rebuild in DB 1
rebuild = redis.Redis(db=1)

# 1. Clear rebuild database
rebuild.flushdb()

# 2. Populate new data
for item in get_all_items():
    rebuild.set(item['key'], item['value'])

# 3. Atomic switch
rebuild.swapdb(0, 1)

# Now DB 0 has new data, DB 1 has old
# If something wrong: swapdb again to rollback
```

## Performance Implications

### Memory Usage

```python
# Each database consumes memory
# With 16 databases, if maxmemory=256MB:
# Average per database: ~16MB

# All databases share maxmemory limit
# Total across all DBs ≤ maxmemory
```

### Key Lookup Performance

```python
# Database selection is O(1)
r.select(1)  # No performance impact

# KEYS operation is O(N)
# Must scan all keys in selected DB
r.keys('*')  # Expensive in large DBs
```

## Best Practices

### 1. Keep Number of Databases Reasonable

```conf
# Good: Use what you need
databases 8

# Avoid: Unnecessary databases waste memory
databases 256

# Typical: 16 is good default
databases 16
```

### 2. Use Meaningful Database Numbers

```python
DATABASES = {
    'prod': 0,
    'staging': 1,
    'dev': 2,
    'cache': 3,
}

# Use constants instead of magic numbers
client = redis.Redis(db=DATABASES['prod'])
```

### 3. Document Database Layout

```python
"""
Redis Database Layout:
- DB 0: Production user sessions
- DB 1: Production cached data
- DB 2: Staging environment
- DB 3: Development environment
- DB 4-7: Reserved for future use
- DB 8-15: Testing/experiments
"""
```

### 4. Consider Namespace Pattern Instead

```python
# Instead of databases, use key prefixes
# More portable, works with clustering

# Equivalent to databases but in single namespace:
r.set('prod:user:123', value)
r.set('test:user:123', value)

# Get all by environment:
r.keys('prod:*')
r.keys('test:*')
```

### 5. Monitor Database Sizes

```redis
# Check which databases have data
INFO keyspace

# Output:
# db0:keys=1000,expires=50,avg_ttl=86400
# db1:keys=500,expires=0,avg_ttl=0
# db2:keys=10,expires=10,avg_ttl=3600
```

## Common Mistakes

### ❌ Assuming Databases are Secure Isolation

```python
# DON'T: Rely on databases for multi-tenant security
client.select(0)  # Tenant A
client.select(1)  # Tenant B - easily accessible

# DO: Use separate Redis instances for true isolation
redis_a = redis.Redis(host='a.redis')
redis_b = redis.Redis(host='b.redis')
```

### ❌ Not Tracking Database Usage

```python
# Bad: Unclear which DB is which
r.select(0)
r.set('key', value)
# Later: What was DB 0 for again?

# Good: Use constants
DB_PROD_CACHE = 0
r.select(DB_PROD_CACHE)
```

### ❌ Over-Complicating with Databases

```python
# Unnecessary: 16 different databases for different types
databases = 16  # Only use 1-2 needed

# Better: Namespace pattern
# prod:user:*, prod:cache:*, prod:session:*
# All in one database, more portable
```

## Next Steps

- [Strings](6-strings.md) - Primary data type
- [Configuration](4-configuration.md) - Database settings
- [Eviction](16-eviction.md) - Shared memory management
- [Persistence](15-persistence.md) - Saving data across databases

## Resources

- **Database Documentation**: https://redis.io/commands/select/
- **SWAPDB**: https://redis.io/commands/swapdb/
- **Key Spaces**: https://redis.io/docs/data-types/

## Summary

- Redis has 16 databases by default, numbered 0-15
- Each database is isolated namespace (SELECT switches)
- All databases share configured maxmemory
- SWAPDB allows atomic database exchange
- Consider namespace pattern for clustering compatibility
- Documents your database layout for team clarity
