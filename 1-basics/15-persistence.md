# Redis Persistence

## Overview

Persistence saves data to disk, enabling recovery after restarts. Redis offers RDB (snapshots) and AOF (append-only file) with different trade-offs between performance and data safety.

## RDB (Snapshot) Persistence

### How RDB Works

```
Memory          Disk
  |             |
  | Data        |
  |             | SAVE/BGSAVE
  |             ├─→ dump.rdb (snapshot)
  |             |
  |        (on restart)
  |←─────────────┤
  | Data loaded  |
  |             |
```

### RDB Configuration

```conf
# redis.conf
# When to trigger snapshot
save 900 1       # 15 mins if 1 key changed
save 300 10      # 5 mins if 10 keys changed
save 60 10000    # 60 secs if 10k keys changed

# Or disable RDB
# save ""

# Snapshot options
stop-writes-on-bgsave-error yes  # Stop if save fails
rdbcompression yes               # Compress snapshot
rdbchecksum yes                  # Include checksum
dbfilename dump.rdb              # Filename
dir /var/lib/redis              # Directory
```

### RDB Commands

```python
import redis

r = redis.Redis()

# Create snapshot (blocks Redis)
r.bgsave()  # Background save (recommended)

# Get last save time
last_save = r.lastsave()
print(f"Last save: {last_save}")

# Check if save in progress
info = r.info('persistence')
print(f"RDB in progress: {info['rdb_bgsave_in_progress']}")
```

### RDB Pros and Cons

**Pros:**
- ✓ Fast recovery (single file)
- ✓ Compact format
- ✓ Low CPU during normal operation
- ✓ Efficient for backups

**Cons:**
- ✗ Data loss since last snapshot
- ✗ Blocking save (older versions)
- ✗ Large file with big datasets

## AOF (Append-Only File) Persistence

### How AOF Works

```
Commands       Disk AOF Log
    |          |
SET key val   ├→ *3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$3\r\nval\r\n
GET key       ├→ *2\r\n$3\r\nGET\r\n$3\r\nkey\r\n
INCR counter  ├→ *2\r\n$4\r\nINCR\r\n$7\r\ncounter\r\n
              |
         (on restart)
         Replay all commands
```

### AOF Configuration

```conf
# redis.conf
# Enable AOF
appendonly yes

# AOF filename
appendfilename "appendonly.aof"

# Fsync policy
appendfsync always     # Fsync every command (safest, slowest)
appendfsync everysec   # Fsync every 1 sec (good balance)
appendfsync no         # Let OS decide (fastest, risky)

# AOF rewrite
auto-aof-rewrite-percentage 100  # Rewrite when 100% larger
auto-aof-rewrite-min-size 64mb   # Min file size to rewrite

# Hybrid persistence (Redis 4.0+)
aof-use-rdb-preamble yes   # RDB header in AOF
```

### AOF Commands

```python
import redis

r = redis.Redis()

# Force AOF rewrite
r.bgrewriteaof()

# Check AOF status
info = r.info('persistence')
print(f"AOF enabled: {info['aof_enabled']}")
print(f"AOF size: {info['aof_current_size']}")
print(f"AOF rewrite in progress: {info['aof_rewrite_in_progress']}")
```

### AOF Pros and Cons

**Pros:**
- ✓ Better data safety (frequent fsync)
- ✓ Readable format
- ✓ Can process partial writes
- ✓ Automatic compaction via rewrite

**Cons:**
- ✗ Slower than RDB
- ✗ Larger file size
- ✗ Slower recovery (replay commands)
- ✗ Higher disk I/O

## Choosing Persistence Strategy

### Strategy 1: RDB Only (Default)

```conf
# Good for: Caches, non-critical data
appendonly no
save 900 1
save 300 10
save 60 10000
```

**When to use:**
- Cache servers (data loss acceptable)
- Non-critical sessions
- High-performance requirements

### Strategy 2: AOF Only

```conf
# Good for: Important data, must preserve
appendonly yes
appendfsync everysec

save ""  # Disable RDB
```

**When to use:**
- Critical data (financial, user info)
- Audit trails
- Transaction logs

### Strategy 3: Hybrid (Recommended)

```conf
# Good for: Balance of safety and performance
save 3600 1        # Less frequent RDB
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
```

**When to use:**
- Most production applications
- Mixed critical and non-critical data
- Good recovery time + data safety

## Real-World Examples

### Example 1: Backup Strategy

```python
import redis
import shutil
from datetime import datetime
import os

def backup_redis():
    """Create Redis backup"""
    r = redis.Redis()
    
    # Force snapshot
    r.bgsave()
    
    # Wait for save to complete
    while r.info('persistence')['rdb_bgsave_in_progress']:
        import time
        time.sleep(0.1)
    
    # Copy dump.rdb
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    backup_file = f"redis_backup_{timestamp}.rdb"
    
    shutil.copy('/var/lib/redis/dump.rdb', backup_file)
    
    print(f"Backup created: {backup_file}")
    
    return backup_file

# Schedule: backup_redis()  # Run daily
```

### Example 2: Verify Persistence Configuration

```python
import redis

def verify_persistence():
    """Check persistence is properly configured"""
    r = redis.Redis(password='password')
    
    config = r.config_get(['save', 'appendonly', 'appendfsync'])
    
    # RDB check
    save_config = config.get('save', '')
    if not save_config or save_config == '':
        print("⚠️  RDB disabled - no snapshot backups")
    else:
        print(f"✓ RDB enabled: {save_config}")
    
    # AOF check
    aof = config.get('appendonly', 'no')
    if aof == 'yes':
        print(f"✓ AOF enabled")
        fsync = config.get('appendfsync', 'everysec')
        print(f"  Fsync: {fsync}")
    else:
        print("⚠️  AOF disabled - no transaction log")
    
    # Hybrid check
    hybrid = r.config_get('aof-use-rdb-preamble')
    print(f"✓ Hybrid RDB+AOF: {hybrid.get('aof-use-rdb-preamble', 'no')}")

verify_persistence()
```

### Example 3: Recovery Scenario

```python
import redis
import os

def disaster_recovery():
    """Restore from backup"""
    # 1. Stop Redis
    # sudo systemctl stop redis
    
    # 2. Restore dump file
    backup_file = "redis_backup_20240101_120000.rdb"
    target_file = "/var/lib/redis/dump.rdb"
    
    shutil.copy(backup_file, target_file)
    os.chown(target_file, redis_user_id, redis_group_id)
    
    # 3. Start Redis
    # sudo systemctl start redis
    
    # 4. Verify
    r = redis.Redis()
    print(f"Redis started, keys: {r.dbsize()}")

# Usage: disaster_recovery()
```

## Persistence Performance

### Comparison Table

| Metric | RDB | AOF | Hybrid |
|--------|-----|-----|--------|
| Write Performance | ★★★★★ | ★★☆☆☆ | ★★★★☆ |
| Disk Space | ★★★★☆ | ★★☆☆☆ | ★★★★☆ |
| Recovery Speed | ★★★★★ | ★★★☆☆ | ★★★★★ |
| Data Safety | ★★★☆☆ | ★★★★★ | ★★★★☆ |
| CPU Usage | High (save) | Medium | Medium |

### Performance Tuning

```conf
# For high throughput (cache)
appendonly no
save ""
# or
save 3600 1  # Daily backup only

# For data safety
appendonly yes
appendfsync always
save 60 1

# For balance
appendonly yes
appendfsync everysec
save 300 10
aof-use-rdb-preamble yes
```

## Monitoring Persistence

### Check Persistence Status

```python
import redis

def monitor_persistence():
    """Monitor persistence operations"""
    r = redis.Redis()
    
    info = r.info('persistence')
    
    print(f"RDB Last Save: {info['rdb_last_save_time']}")
    print(f"RDB Changes Since Save: {info['rdb_changes_since_last_save']}")
    print(f"RDB BGSAVE in progress: {info['rdb_bgsave_in_progress']}")
    print(f"RDB BGSAVE last status: {info['rdb_last_bgsave_status']}")
    
    print(f"AOF Enabled: {info['aof_enabled']}")
    print(f"AOF Rewrite in progress: {info['aof_rewrite_in_progress']}")
    print(f"AOF Rewrite last status: {info['aof_last_rewrite_status']}")

monitor_persistence()
```

## Best Practices

### 1. Use RDB + AOF Hybrid

```conf
# Best of both worlds
aof-use-rdb-preamble yes
appendonly yes
appendfsync everysec

# Still backup snapshots
save 3600 1
```

### 2. Monitor Rewrite Size

```python
# AOF files grow over time
# Rewrite prevents unbounded growth
# Monitor progress:

r = redis.Redis()
info = r.info('persistence')
print(f"AOF size: {info['aof_current_size']}")
print(f"AOF base size: {info['aof_base_size']}")

# If growing too fast, adjust:
# auto-aof-rewrite-percentage 100
# auto-aof-rewrite-min-size 64mb
```

### 3. Separate Persistence Disk

```conf
# Put dump.rdb and appendonly.aof on separate disk
dir /mnt/redis-data  # Fast SSD

# Also backup to different device
# Prevent single disk failure = total loss
```

### 4. Regular Backup Testing

```python
def test_restore():
    """Periodically test restore process"""
    # 1. Create backup
    r = redis.Redis()
    r.bgsave()
    
    # 2. Wait for completion
    # 3. Copy backup to test system
    # 4. Start Redis from backup
    # 5. Verify data integrity
    
    print("Backup restore test successful")

# Run monthly
test_restore()
```

## Common Issues

### Issue 1: "Background save in progress"

```python
# Multiple BGSAVE commands
r.bgsave()
r.bgsave()  # Error: Background save already in progress

# Solution: Check status first
if not r.info('persistence')['rdb_bgsave_in_progress']:
    r.bgsave()
```

### Issue 2: AOF File Too Large

```python
# Solution: Force rewrite
r.bgrewriteaof()

# Or adjust thresholds:
# auto-aof-rewrite-percentage 100
# auto-aof-rewrite-min-size 64mb
```

### Issue 3: Slow Recovery

```python
# Large RDB or AOF files take time to load
# Solutions:
# 1. Hybrid mode (faster)
# 2. Distribute data across multiple instances
# 3. Pre-warm cache after restore
```

## Next Steps

- [Configuration](4-configuration.md) - Tuning persistence
- [Security](14-security.md) - Backup security
- [Eviction](16-eviction.md) - Memory management

## Resources

- **Persistence**: https://redis.io/docs/management/persistence/
- **RDB**: https://redis.io/docs/management/persistence/#rdb
- **AOF**: https://redis.io/docs/management/persistence/#aof

## Summary

- RDB: Fast snapshots, data loss risk
- AOF: Safe transactions, slower recovery
- Hybrid: Best of both (Redis 4.0+)
- Use for production (except cache-only)
- Test restore regularly
- Monitor persistence operations
