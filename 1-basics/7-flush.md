# Redis Flush Operations

## Overview

Flush operations delete all keys from databases. Redis provides FLUSHDB (current database) and FLUSHALL (all databases). Critical operation requiring caution in production.

## Flush Commands

### FLUSHDB - Flush Current Database

Delete all keys in the currently selected database:

```redis
# Synchronous flush (blocks Redis)
FLUSHDB

# Asynchronous flush (non-blocking)
FLUSHDB ASYNC

# With confirmation
FLUSHDB SYNC   # Force synchronous
```

### FLUSHALL - Flush All Databases

Delete all keys in all databases:

```redis
# Synchronous flush (all databases)
FLUSHALL

# Asynchronous flush (all databases)
FLUSHALL ASYNC

# Force synchronous
FLUSHALL SYNC
```

## Command Details

### Synchronous vs Asynchronous

**SYNC (Default)**
- Blocks Redis thread during deletion
- All clients blocked until complete
- May cause 100ms-seconds latency for large datasets
- Used when data loss acceptable and speed needed

```redis
# Blocks for duration of deletion
FLUSHDB
# If 1M keys: ~500ms-1s delay
```

**ASYNC (Redis 4.0+)**
- Deletes in background thread
- Redis continues serving requests
- Recommended for production
- May take longer but non-blocking

```redis
# Returns immediately
FLUSHDB ASYNC
# Deletion happens in background
```

## Use Cases

### 1. Development Testing

```python
import redis

r = redis.Redis()

# Before each test
r.flushdb()

# Run test
r.set('test_key', 'value')
assert r.get('test_key') == 'value'

# Clean up after test
r.flushdb()
```

### 2. Data Reset in Staging

```python
def reset_staging_cache():
    # Connect to staging Redis
    r = redis.Redis(host='staging-redis')
    
    # Clear all cache
    r.flushdb()
    
    print("Staging cache cleared")

# Run periodically
reset_staging_cache()
```

### 3. Multi-Environment Cleanup

```python
def clear_old_cache(days=30):
    """Clear cache older than N days"""
    r = redis.Redis()
    
    # Get all keys
    keys = r.keys('cache:*')
    
    # Delete if old
    for key in keys:
        ttl = r.ttl(key)
        if ttl > days * 86400:  # Over 30 days TTL
            r.delete(key)

def emergency_cache_clear():
    """Clear everything on emergency"""
    r = redis.Redis()
    r.flushdb(async_=True)  # Non-blocking
```

### 4. Database Swap Pattern

```python
def reload_data_atomically():
    """Reload data without losing connections"""
    r = redis.Redis()
    
    # Stage data in DB 1 while DB 0 serves
    r.select(1)
    r.flushdb()  # Clear staging
    
    # Load new data into DB 1
    load_data_from_source(r)
    
    # Atomic swap
    r.swapdb(0, 1)
    
    # DB 0 now has new data
```

## Patterns and Best Practices

### Pattern 1: Safe FLUSHDB in Production

```python
import redis
import logging

logger = logging.getLogger(__name__)

def safe_flush_database(confirm=True):
    """Safely flush database with confirmation"""
    if confirm:
        # Require explicit confirmation
        response = input("Are you sure? Type 'YES' to confirm: ")
        if response != "YES":
            logger.warning("FLUSHDB cancelled by user")
            return False
    
    r = redis.Redis()
    
    # Use async to avoid blocking
    r.flushdb(async_=True)
    
    logger.warning("Database flushed asynchronously")
    return True
```

### Pattern 2: Backup Before Flush

```python
import redis
import json
from datetime import datetime

def backup_and_flush():
    """Backup database before flushing"""
    r = redis.Redis()
    
    # 1. Backup all keys
    backup = {}
    for key in r.keys('*'):
        backup[key] = r.dump(key)
    
    # 2. Save backup
    backup_file = f"backup-{datetime.now().isoformat()}.json"
    with open(backup_file, 'w') as f:
        json.dump(backup, f)
    
    print(f"Backup saved to {backup_file}")
    
    # 3. Flush
    r.flushdb(async_=True)
    
    print("Database flushed")
```

### Pattern 3: Conditional Flush

```python
import redis
import os

def conditional_flush():
    """Only flush in specific environments"""
    env = os.getenv('ENVIRONMENT', 'prod')
    
    r = redis.Redis()
    
    if env == 'dev':
        # Safe in development
        r.flushdb()
    elif env == 'staging':
        # Confirm in staging
        confirm = input("Flush staging? (yes/no): ")
        if confirm == 'yes':
            r.flushdb(async_=True)
    elif env == 'prod':
        # Strongly discourage in production
        raise Exception("Cannot flush production directly!")
```

### Pattern 4: Selective Data Retention

```python
import redis

def flush_with_protection():
    """Flush but keep important data"""
    r = redis.Redis()
    
    # Important keys to keep
    PROTECTED_KEYS = {
        'config:*',      # Configuration
        'user:premium:*', # Premium users
    }
    
    # Get all keys
    all_keys = r.keys('*')
    
    # Backup protected keys
    protected_data = {}
    for key in all_keys:
        if any(key.startswith(pattern.rstrip('*')) for pattern in PROTECTED_KEYS):
            protected_data[key] = r.dump(key)
    
    # Flush everything
    r.flushdb()
    
    # Restore protected keys
    for key, data in protected_data.items():
        r.restore(key, 0, data)
    
    print(f"Flushed, but kept {len(protected_data)} protected keys")
```

## Production Considerations

### 1. Use ASYNC in Production

```python
# Good: Non-blocking
r.flushdb(async_=True)

# Avoid: Blocking in production
r.flushdb()  # Will block all clients!
```

### 2. Schedule Off-Peak

```python
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()

def scheduled_flush():
    r = redis.Redis()
    r.flushdb(async_=True)

# Flush at 2 AM (low traffic)
scheduler.add_job(scheduled_flush, 'cron', hour=2, minute=0)
scheduler.start()
```

### 3. Monitor Flush Operations

```python
import redis
import time

def monitored_flush():
    """Flush with monitoring"""
    r = redis.Redis()
    
    # Get initial stats
    info_before = r.info('stats')
    
    # Flush asynchronously
    r.flushdb(async_=True)
    
    # Monitor progress
    start_time = time.time()
    while True:
        info = r.info('stats')
        elapsed = time.time() - start_time
        
        # Check if flush complete (would see other operations resume)
        if elapsed > 5:  # Check for 5 seconds
            break
        
        time.sleep(0.1)
    
    print(f"Flush completed in {elapsed:.2f}s")
```

### 4. Prevent Accidental Flush

```conf
# In redis.conf - Disable FLUSHDB/FLUSHALL
rename-command FLUSHDB ""
rename-command FLUSHALL ""

# Or rename to something harder
rename-command FLUSHDB "XYZZY_FLUSHDB_XYZZY"
rename-command FLUSHALL "XYZZY_FLUSHALL_XYZZY"
```

```python
# Then access via renamed command
def flush_renamed():
    r = redis.Redis()
    r.execute_command("XYZZY_FLUSHDB_XYZZY")
```

## Performance Implications

### Time Complexity

| Operation | Complexity |
|-----------|-----------|
| FLUSHDB | O(N) where N = number of keys |
| FLUSHALL | O(N) across all databases |

### Blocking Time Estimates

```
100 keys:       ~1ms
10,000 keys:    ~10ms
1,000,000 keys: ~500ms-1s
10,000,000 keys: ~5-10s (synchronous)
```

### Memory Impact

- Synchronous: Blocks during cleanup
- Asynchronous: Uses background thread (small CPU cost)

## Common Mistakes

### ❌ FLUSHALL by Mistake

```python
# BAD: Easy to confuse commands
r.flushall()  # Oops! Deleted everything

# Good: Use FLUSHDB
r.flushdb()  # Only current database

# Better: Have confirmations
if confirm_operation("This will delete all data"):
    r.flushdb(async_=True)
```

### ❌ Synchronous Flush in Production

```python
# Bad: Blocks all clients
r.flushdb()  # 500ms-1s+ delay for all operations

# Good: Non-blocking
r.flushdb(async_=True)
```

### ❌ No Backup Before Flush

```python
# Bad: No recovery possible
r.flushdb()  # Data lost forever

# Good: Backup first
backup_data()
r.flushdb(async_=True)
```

### ❌ Forgetting Multi-Database Impact

```python
# Bad: Forgot you're not flushing DB 0
r.select(5)
r.flushdb()  # Only flushes DB 5

# Good: Flush specific or all
r.select(0)
r.flushdb()  # Flush specific

r.flushall()  # Flush all if intended
```

## Safety Checklist

Before flushing:
- [ ] Confirm correct environment (not production)
- [ ] Backup important data
- [ ] Notify team members
- [ ] Use ASYNC in production
- [ ] Schedule during maintenance window
- [ ] Monitor for issues
- [ ] Have rollback plan

## Next Steps

- [Database Selection](5-database.md) - Managing multiple databases
- [Persistence](15-persistence.md) - RDB/AOF backup strategies
- [Configuration](4-configuration.md) - Preventing accidental flush
- [Transactions](9-transaction.md) - Atomic operations

## Resources

- **FLUSHDB**: https://redis.io/commands/flushdb/
- **FLUSHALL**: https://redis.io/commands/flushall/
- **Safety**: https://redis.io/docs/management/troubleshooting/

## Summary

- FLUSHDB deletes keys in current database (O(N))
- FLUSHALL deletes keys in all databases (O(N))
- Use ASYNC in production to avoid blocking
- Always backup before flushing
- Prevent accidental flush via renaming commands
- Schedule during maintenance windows
- Confirm operation before executing
