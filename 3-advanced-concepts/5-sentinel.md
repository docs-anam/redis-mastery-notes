# Sentinel - Automatic Failover and Monitoring

## Overview

Redis Sentinel monitors master-replica deployments and automatically promotes replicas to master if the master fails.

### Architecture
```
Clients connect to Sentinel
          |
          v
┌─────────────────────────┐
│    Sentinel Cluster     │
│  (3-5 sentinels)        │
│  Monitors & Failover    │
└─────────────────────────┘
     |              |
     v              v
   MASTER       REPLICAS
   |_________________________|
   Automatic promotion on master failure
```

### Key Benefits

✅ **Automatic Failover**: Promotes replica without manual intervention
✅ **Monitoring**: Continuously watches master health
✅ **Client Discovery**: Provides dynamic master address
✅ **Configuration Management**: Updates config on failover
✅ **Notifications**: Alerts on failure/promotion events

### When to Use

✅ **Good For**:
- Production systems requiring high availability
- Zero-downtime deployments
- Automatic recovery from failures
- Critical services

❌ **Not For**:
- Development/testing
- Single server deployments
- Simple caching layers

---

## Core Concepts

### Quorum

Minimum sentinels needed to declare master down:
```
5 Sentinels -> 3+ needed for failover
3 Sentinels -> 2+ needed for failover
1 Sentinel -> Can't failover safely
```

### Failure Detection

```
Master            Sentinels
  |                |
  +-- No heartbeat for 30s --┤
                   |
                   +-- 3 sentinels agree down
                   |
                   v
               FAILOVER STARTS
```

### Configuration Propagation

```
Master:6379 fails
    |
    v
Sentinel detects failure
    |
    v
Sentinel promotes replica:6380 -> new master
    |
    v
Updates sentinel.conf with new master
    |
    v
Clients discover new master via Sentinel
```

---

## Setup

### Step 1: Create sentinel.conf

```
# /etc/redis/sentinel.conf
port 26379
daemonize yes
pidfile /var/run/redis-sentinel.pid
logfile /var/log/redis-sentinel.log

# Monitor master
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000

# Notifications
sentinel notification-script mymaster /path/to/notification.sh
sentinel client-reconfig-script mymaster /path/to/reconfig.sh
```

### Step 2: Start Sentinel

```bash
redis-sentinel /etc/redis/sentinel.conf
```

### Step 3: Verify Sentinel

```bash
redis-cli -p 26379
> SENTINEL masters
> SENTINEL slaves mymaster
> SENTINEL sentinels mymaster
```

---

## Commands Reference

### SENTINEL MASTERS
List all monitored masters.

```redis
SENTINEL masters
# Returns: List of master info
```

```python
import redis

sentinel = redis.Sentinel([('localhost', 26379)])
masters = sentinel.execute_command('SENTINEL', 'masters')
```

### SENTINEL SLAVES
List replicas of a master.

```redis
SENTINEL slaves mymaster
```

```python
slaves = sentinel.execute_command('SENTINEL', 'slaves', 'mymaster')
```

### SENTINEL GET-MASTER-ADDR-BY-NAME
Get master address for named service.

```redis
SENTINEL GET-MASTER-ADDR-BY-NAME mymaster
# Returns: [ip, port]
```

```python
master_addr = sentinel.execute_command(
    'SENTINEL', 'GET-MASTER-ADDR-BY-NAME', 'mymaster'
)
print(f"Master: {master_addr[0]}:{master_addr[1]}")
```

### SENTINEL FAILOVER
Manually trigger failover.

```redis
SENTINEL failover mymaster
```

```python
sentinel.execute_command('SENTINEL', 'failover', 'mymaster')
```

---

## Practical Examples

### Example 1: Sentinel Client Connection

```python
import redis
from redis.sentinel import Sentinel

class SentinelCluster:
    def __init__(self, sentinel_hosts):
        # List of (host, port) tuples
        self.sentinel = Sentinel(sentinel_hosts)
    
    def get_master(self):
        """Get connection to master"""
        return self.sentinel.master_for('mymaster')
    
    def get_slave(self):
        """Get connection to replica"""
        return self.sentinel.slave_for('mymaster')
    
    def write(self, key, value):
        """Write to master"""
        master = self.get_master()
        master.set(key, value)
    
    def read(self, key):
        """Read from replica"""
        slave = self.get_slave()
        return slave.get(key)
    
    def check_status(self):
        """Check cluster status"""
        info = self.sentinel.master_for('mymaster').info()
        return {
            'role': info.get('role'),
            'connected_slaves': info.get('connected_slaves'),
            'uptime_seconds': info.get('uptime_in_seconds')
        }

# Usage
cluster = SentinelCluster([
    ('sentinel1', 26379),
    ('sentinel2', 26379),
    ('sentinel3', 26379)
])

cluster.write('key', 'value')
print(cluster.read('key'))
print(cluster.check_status())
```

### Example 2: Health Monitoring

```python
import redis
from redis.sentinel import Sentinel
import time

class HealthMonitor:
    def __init__(self, sentinel_hosts):
        self.sentinel = Sentinel(sentinel_hosts)
    
    def monitor_masters(self, interval=5):
        """Monitor master health"""
        while True:
            try:
                masters = self.sentinel.execute_command('SENTINEL', 'masters')
                for master in masters:
                    name = master[b'name'].decode()
                    flags = master[b'flags'].decode()
                    
                    print(f"Master {name}: {flags}")
                    
                    if b'disconnected' in flags or b'failover' in flags:
                        self.alert_failover(name)
                
                time.sleep(interval)
            except redis.ConnectionError:
                print("Lost connection to Sentinel")
                time.sleep(interval)
    
    def alert_failover(self, master_name):
        """Alert on failover"""
        print(f"⚠️  Master {master_name} failing over!")

# Usage
monitor = HealthMonitor([('localhost', 26379)])
# monitor.monitor_masters()  # Run in background
```

### Example 3: Automatic Failover Recovery

```python
import redis
from redis.sentinel import Sentinel
import time

class FailoverHandler:
    def __init__(self, sentinel_hosts, service_name):
        self.sentinel = Sentinel(sentinel_hosts)
        self.service_name = service_name
        self._retries = 0
        self._max_retries = 3
    
    def execute_with_failover(self, operation, *args):
        """Execute with automatic failover recovery"""
        while self._retries < self._max_retries:
            try:
                master = self.sentinel.master_for(self.service_name)
                return operation(master, *args)
            except redis.ConnectionError:
                self._retries += 1
                if self._retries < self._max_retries:
                    wait_time = 2 ** self._retries  # Exponential backoff
                    print(f"Failover in progress, retrying in {wait_time}s...")
                    time.sleep(wait_time)
                else:
                    raise
    
    def set_key(self, key, value):
        """SET with failover handling"""
        def op(master, k, v):
            master.set(k, v)
        
        self.execute_with_failover(op, key, value)
    
    def get_key(self, key):
        """GET with failover handling"""
        def op(master, k):
            return master.get(k)
        
        return self.execute_with_failover(op, key)

# Usage
handler = FailoverHandler(
    [('localhost', 26379)], 
    'mymaster'
)

handler.set_key('critical_data', 'important_value')
print(handler.get_key('critical_data'))
```

---

## Best Practices

### DO: Deploy Odd Number of Sentinels

```
✅ GOOD:
- 3 sentinels (2 needed for quorum)
- 5 sentinels (3 needed for quorum)
- 7 sentinels (4 needed for quorum)

❌ BAD:
- 2 sentinels (can't reach quorum if 1 fails)
- 4 sentinels (can't reach quorum if 2 fail)
```

### DO: Separate Sentinel from Master/Replica

```
✅ GOOD: Sentinel on different hardware
Sentinel1 (dedicated box)
Sentinel2 (dedicated box)
Sentinel3 (dedicated box)
Master (dedicated box)
Replica1 (dedicated box)
Replica2 (dedicated box)

❌ BAD: All on same box
(Box failure takes everything down)
```

### DO: Use Notification Scripts

```python
# notification.sh
#!/bin/bash
# Alert on failover
MASTER=$1
ROLE=$2
STATE=$3

if [ "$STATE" = "failover-end" ]; then
    curl -X POST http://alerts.example.com/failover \
        -d "master=$MASTER"
fi
```

### DON'T: Set Min Replicas to 0

```
# ❌ BAD: Allows failover with no replicas
sentinel parallel-syncs mymaster 0

# ✅ GOOD: Ensure at least 1 replica
sentinel parallel-syncs mymaster 1
```

### DON'T: Use High Down-After Time

```
# ❌ BAD: 60 second delay before failover
sentinel down-after-milliseconds mymaster 60000

# ✅ GOOD: 5 second response
sentinel down-after-milliseconds mymaster 5000
```

---

## Common Mistakes

### Mistake 1: Insufficient Sentinels

**Problem**: Single Sentinel failure blocks failover
```
# ❌ BAD: Only 1 Sentinel
3 Sentinels needed, 1 available
Failover impossible
```

**Solution**: Deploy 3-5 Sentinels
```
# ✅ GOOD: Minimum 3
sentinel monitor mymaster ... 2
```

### Mistake 2: Sentinels on Master Box

**Problem**: Box failure takes out both
```
# ❌ BAD: All on master box
Box failure -> Master down, Sentinels down
Can't failover (no Sentinels running)
```

**Solution**: Separate hardware
```
# ✅ GOOD: Distributed
Sentinel1 on box A
Sentinel2 on box B
Sentinel3 on box C
```

### Mistake 3: Not Handling Failover

**Problem**: Application doesn't know about new master
```python
# ❌ BAD: Hardcoded master address
r = redis.Redis(host='master.example.com', port=6379)

# Failover happens
# Connection fails, no recovery
```

**Solution**: Use Sentinel client
```python
# ✅ GOOD: Dynamic discovery
sentinel = Sentinel([...])
r = sentinel.master_for('mymaster')

# Failover happens
# Client automatically discovers new master
```

### Mistake 4: Quorum Problems

**Problem**: Split brain or wrong quorum settings
```
# ❌ BAD: More sentinels than needed
sentinel monitor mymaster ... 5
# But only 3 sentinels deployed

Failover requires 5, but only 3 available!
```

**Solution**: Set quorum = (n_sentinels / 2) + 1
```
# ✅ GOOD: 3 sentinels, quorum 2
sentinel monitor mymaster ... 2
```

### Mistake 5: Long Failover Times

**Problem**: Slow detection and failover
```
# ❌ BAD: 30 second detection
sentinel down-after-milliseconds mymaster 30000
# Plus 30 second failover timeout
sentinel failover-timeout mymaster 30000
# Total: 60 seconds downtime
```

**Solution**: Aggressive timeouts
```
# ✅ GOOD: Quick detection
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
# Total: ~15 seconds downtime
```

---

## Monitoring Checklist

Before production:

```
□ 3+ Sentinels deployed
□ Sentinels on separate hardware
□ Quorum correctly set
□ Master reachable from all sentinels
□ Replicas reachable from all sentinels
□ Network connectivity tested
□ Failover tested manually
□ Client library supports Sentinel
□ Monitoring alerts configured
□ Runbooks documented
```

---

## Next Steps

1. **Learn [Cluster](6-cluster.md)** for horizontal scaling
2. **Explore [Performance](../4-performance/1-intro.md)** tuning
3. **Study [Monitoring](../1-basics/11-server-information.md)** tools
4. **Review [Persistence](../1-basics/15-persistence.md)** best practices

## Resources

- [Redis Sentinel Official](https://redis.io/topics/sentinel)
- [Sentinel Commands](https://redis.io/commands/sentinel-masters)
- [Sentinel Configuration](https://redis.io/topics/sentinel#configuration)
- [Sentinel Events](https://redis.io/topics/sentinel#notification-scripts)
