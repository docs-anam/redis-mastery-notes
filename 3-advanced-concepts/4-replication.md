# Replication - High Availability and Read Scaling

## Table of Contents
1. [Overview](#overview)
2. [Core Concepts](#core-concepts)
3. [Setup and Configuration](#setup-and-configuration)
4. [Commands Reference](#commands-reference)
5. [Practical Examples](#practical-examples)
6. [Real-World Patterns](#real-world-patterns)
7. [Performance Characteristics](#performance-characteristics)
8. [Best Practices](#best-practices)
9. [Common Mistakes](#common-mistakes)

---

## Overview

Replication provides data redundancy and read scaling by synchronizing data from a master to one or more replicas (slaves).

### Architecture
```
Clients (Write)
     |
     v
  MASTER
     |
     +-------+-------+
     |       |       |
     v       v       v
  REPLICA REPLICA REPLICA
     ^       ^       ^
     |       |       |
 Clients (Read-Only)
```

### Key Benefits

✅ **Data Redundancy**: Data persisted on multiple servers
✅ **Read Scaling**: Distribute read queries across replicas
✅ **Backup**: Easy backup from replica without blocking master
✅ **Analytics**: Run heavy queries on replica
✅ **Failover**: Basis for automatic failover with Sentinel

### Limitations

❌ **Eventual Consistency**: Replicas lag behind master
❌ **No Automatic Failover**: Need Sentinel for this
❌ **Single Master Writes**: All writes to master
❌ **Replication Lag**: May read stale data from replica

---

## Core Concepts

### Replication Methods

**Full Resynchronization**
```
Master sends entire dataset snapshot to replica
- Used for initial sync
- Slower, more data
- ~100-500ms for 1GB
```

**Partial Resynchronization**
```
Master sends only new commands
- Used after disconnection
- Much faster
- Requires backlog buffer
```

### Replication Lag

```
Master      Replica
  GET x     (waiting)
  SET y     (executing)
  INCR z    (executing GET x)
```

Replica lags behind due to:
- Network latency
- Command processing time
- Replica write speed

### Replication ID and Offset

```
Replication ID: Unique ID per master (changes on REPLICAOF)
Offset: Number of bytes replicated
- Master tracks its offset
- Replica tracks received offset
- Difference = how far behind
```

---

## Setup and Configuration

### Step 1: Configure Master

```
# /etc/redis/redis.conf on master
port 6379
bind 0.0.0.0
requirepass masterpass
```

### Step 2: Configure Replica

```
# /etc/redis/replica.conf
port 6380
bind 0.0.0.0
replicaof 127.0.0.1 6379
masterauth masterpass
```

### Step 3: Activate Replication

```bash
# On replica server
redis-server /etc/redis/replica.conf
```

Replica automatically syncs with master!

---

## Commands Reference

### REPLICAOF
Configure the server as a replica of another instance.

```redis
REPLICAOF host port
REPLICAOF NO ONE  # Stop replication
```

```python
import redis

r = redis.Redis(host='localhost', port=6380)

# Make this server a replica
r.replicaof('master.example.com', 6379)

# Stop replication
r.replicaof(None, None)
```

### INFO replication
Get replication status.

```redis
INFO replication
# Shows: role, connected_slaves, replication_offset, etc.
```

```python
info = r.info('replication')
print(f"Role: {info['role']}")
print(f"Connected slaves: {info.get('connected_slaves', 0)}")
print(f"Replication offset: {info['replication_offset']}")
```

### ROLE
Get current server role.

```redis
ROLE
# Returns: master|slave, port, replication_offset
```

```python
role_info = r.execute_command('ROLE')
print(role_info)
# Output: (b'master', 0, [(b'127.0.0.1', 6380, b'123')])
```

### SYNC / PSYNC
Initiate synchronization (automatic, rarely used directly).

```redis
SYNC          # Full resynchronization
PSYNC ? -1    # Partial or full
```

---

## Practical Examples

### Example 1: Basic Master-Replica Setup

```python
import redis
import time

class ReplicationManager:
    def __init__(self, master_host, master_port, replica_host, replica_port):
        self.master = redis.Redis(host=master_host, port=master_port)
        self.replica = redis.Redis(host=replica_host, port=replica_port)
    
    def setup_replication(self):
        """Configure replica to follow master"""
        # Make replica follow master
        self.replica.replicaof(master_host, master_port)
        
        # Wait for sync to complete
        time.sleep(2)
        
        # Verify
        info = self.replica.info('replication')
        if info['role'] == 'slave':
            print(f"✓ Replication active")
            print(f"  Lag: {info.get('slave_repl_offset', 0) - info['master_repl_offset']}")
            return True
        return False
    
    def write_to_master(self, key, value):
        """Write to master"""
        self.master.set(key, value)
        print(f"✓ Written to master: {key}={value}")
    
    def read_from_replica(self, key):
        """Read from replica"""
        value = self.replica.get(key)
        return value.decode() if value else None
    
    def check_lag(self):
        """Get replication lag in bytes"""
        master_info = self.master.info('replication')
        replica_info = self.replica.info('replication')
        
        master_offset = master_info['replication_offset']
        replica_offset = replica_info['slave_repl_offset']
        lag = master_offset - replica_offset
        
        print(f"Replication lag: {lag} bytes")
        return lag

# Usage
manager = ReplicationManager('localhost', 6379, 'localhost', 6380)
manager.setup_replication()

manager.write_to_master('user:1:name', 'Alice')
time.sleep(0.5)
print(f"Read from replica: {manager.read_from_replica('user:1:name')}")

manager.check_lag()
```

### Example 2: Read Scaling with Multiple Replicas

```python
import redis
from typing import List

class ReadScalableCluster:
    def __init__(self, master_host, master_port, replica_hosts_ports: List[tuple]):
        self.master = redis.Redis(host=master_host, port=master_port)
        self.replicas = [
            redis.Redis(host=host, port=port) 
            for host, port in replica_hosts_ports
        ]
        self._current_replica = 0
    
    def write(self, key, value):
        """All writes to master"""
        self.master.set(key, value)
    
    def read(self, key, prefer_replica=True):
        """Read-only from replicas when possible"""
        if not prefer_replica or not self.replicas:
            return self.master.get(key)
        
        # Round-robin across replicas
        replica = self.replicas[self._current_replica % len(self.replicas)]
        self._current_replica += 1
        
        return replica.get(key)
    
    def bulk_read(self, keys, use_replica=True):
        """Read multiple keys from replica"""
        source = self.replicas[0] if use_replica and self.replicas else self.master
        return {key: source.get(key) for key in keys}

# Usage
cluster = ReadScalableCluster(
    'localhost', 6379,
    [('localhost', 6380), ('localhost', 6381), ('localhost', 6382)]
)

# Heavy write
for i in range(1000):
    cluster.write(f'key:{i}', f'value:{i}')

# Distributed reads
for i in range(100):
    value = cluster.read(f'key:{i}')

print("✓ Distributed read operations across replicas")
```

### Example 3: Backup with Replica

```python
import redis
import subprocess
import time

class SafeBackup:
    def __init__(self, backup_replica_host, backup_replica_port):
        self.backup_replica = redis.Redis(
            host=backup_replica_host, 
            port=backup_replica_port
        )
    
    def create_backup(self, backup_path):
        """Create backup from replica without blocking master"""
        # Verify we're on replica
        info = self.backup_replica.info('replication')
        if info['role'] != 'slave':
            raise Exception("Must backup from replica only")
        
        # Trigger BGSAVE on replica
        self.backup_replica.bgsave()
        
        # Wait for completion
        while self.backup_replica.lastsave() < time.time() - 1:
            time.sleep(0.5)
        
        # Copy the RDB file
        print("✓ Backup completed from replica")

# Usage
backup = SafeBackup('localhost', 6380)
backup.create_backup('/backups/redis.rdb')
```

---

## Real-World Patterns

### Pattern 1: Write-Master, Read-Replica

```python
import redis
import time

class WriteReadSplitting:
    def __init__(self):
        self.write_db = redis.Redis(host='master', port=6379)
        self.read_db = redis.Redis(host='replica', port=6380)
    
    def set_user(self, user_id, data):
        """Write to master"""
        self.write_db.hset(f'user:{user_id}', mapping=data)
    
    def get_user(self, user_id):
        """Read from replica (might be slightly stale)"""
        return self.read_db.hgetall(f'user:{user_id}')
    
    def get_user_latest(self, user_id):
        """Read from master (always fresh)"""
        return self.write_db.hgetall(f'user:{user_id}')

# Usage
splitter = WriteReadSplitting()
splitter.set_user(123, {'name': 'Alice', 'email': 'alice@example.com'})

# Slightly stale read
user = splitter.get_user(123)

# Always fresh read
user_latest = splitter.get_user_latest(123)
```

### Pattern 2: Cascade Replication

```
Master
  |
  +-- Replica1
       |
       +-- Replica2 (replica of Replica1)
            |
            +-- Replica3 (replica of Replica2)
```

Reduces master load for many replicas.

```python
# Configure Replica2 to replicate from Replica1
replica2.replicaof('replica1_host', 6380)
```

---

## Performance Characteristics

### Replication Speed

```
Network                Replication Speed
────────────────────────────────────
100 Mbps              ~12 MB/s
Gigabit              ~100 MB/s
High-Speed LAN       ~500 MB/s
```

### Sync Times (for 1GB dataset)

```
Network         Full Sync    Partial Sync
──────────────────────────────────────
Local LAN       50-100ms    1-5ms
Same DC         200-500ms   5-20ms
Different DC    2-5s        20-50ms
```

---

## Best Practices

### DO: Monitor Replication Lag

```python
# ✅ GOOD: Regular monitoring
def check_replication_health(master, replica):
    master_info = master.info('replication')
    replica_info = replica.info('replication')
    
    lag = master_info['replication_offset'] - replica_info['slave_repl_offset']
    if lag > 1000000:  # 1MB lag
        alert("High replication lag!")
```

### DO: Use Replicas for Backups Only

```python
# ✅ GOOD: Backup from replica
backup_replica.bgsave()

# ❌ BAD: Run heavy queries on replica that's still syncing
```

### DO: Write to Master, Read from Replica

```python
# ✅ GOOD: Clear separation
master.set(key, value)
value = replica.get(key)

# ❌ BAD: Reading critical data from replica
critical_data = replica.get(critical_key)
```

### DON'T: Write to Replica

```python
# ❌ BAD: Data will be overwritten by replication
replica.set(key, value)

# ✅ GOOD: Write to master
master.set(key, value)
```

### DON'T: Ignore Replication Lag

```python
# ❌ BAD: Data visibility issue
user.set('name', 'Bob')
name = replica.get('name')  # Might not be 'Bob' yet!

# ✅ GOOD: Read from master for consistency
name = master.get('name')  # Always 'Bob'
```

---

## Common Mistakes

### Mistake 1: Writing to Replica

**Problem**: Data loss on failover
```python
# ❌ BAD: Writing to replica
replica.set('balance', 1000)
# This gets overwritten when master sends update
```

**Solution**: Only write to master
```python
# ✅ GOOD: Write to master
master.set('balance', 1000)
# Safely replicated to all replicas
```

### Mistake 2: Not Monitoring Lag

**Problem**: Stale data read without knowing
```python
# ❌ BAD: No lag checking
value = replica.get('key')
# Might be 5 seconds old!
```

**Solution**: Monitor lag
```python
# ✅ GOOD: Check lag before reading
lag = master_offset - replica_offset
if lag > threshold:
    read_from_master()
else:
    read_from_replica()
```

### Mistake 3: Too Many Replicas

**Problem**: Master overwhelmed by replication
```
Master syncing to 50 replicas
= 50x network overhead
```

**Solution**: Use cascade replication
```
Master -> Replica1 -> Replica2 -> Replica3
       -> Replica4 -> Replica5
```

### Mistake 4: Replication with AOF

**Problem**: Conflicting persistence and replication
```
Master: RDB + AOF replication
Replica: AOF conflicts with replication
```

**Solution**: Use RDB for replication
```python
# ✅ GOOD: RDB-based replication
save 900 1  # RDB snapshot
```

### Mistake 5: Not Handling Failures

**Problem**: Replica failure breaks replication
```python
# ❌ BAD: No error handling
replica.execute_command('SYNC')
# If network fails, just gives up
```

**Solution**: Auto-reconnect with backoff
```python
# ✅ GOOD: Automatic reconnection
while True:
    try:
        replica.replicaof(master_host, master_port)
        break
    except redis.ConnectionError:
        time.sleep(5)  # Backoff
```

---

## Performance Comparison

| Aspect | Master Only | Master + 1 Replica | Master + 3 Replicas |
|--------|------------|------------------|-------------------|
| Write throughput | 100% | 85-90% | 60-70% |
| Read throughput | 100% | 200% | 300%+ |
| Data redundancy | None | Good | Excellent |
| Setup complexity | Simple | Medium | Complex |

---

## Next Steps

1. **Learn [Sentinel](5-sentinel.md)** for automatic failover
2. **Explore [Cluster](6-cluster.md)** for horizontal scaling
3. **Study [Performance](../4-performance/1-intro.md)** for optimization
4. **Read [Persistence](../1-basics/15-persistence.md)** for durability

## Resources

- [Redis Replication Official](https://redis.io/topics/replication)
- [Replication Offset Tracking](https://redis.io/topics/replication#replication-offset)
- [Backlog Buffer](https://redis.io/topics/replication#partial-resynchronizations)
